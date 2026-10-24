# البروتوكول

وثيقة مرجعية لبروتوكول Scone: الصيغة الثنائية، الترميز
القانوني، المعاملات، سجلات DNS، الكتل، رسائل P2P، الحدود
وتوافق الإصدارات. التنفيذ المرجعي هو الـ crate
[`scone-protocol`](../../../crates/scone-protocol).

العناصر الموسومة بـ **(مؤقت)** ليست نهائية.

## إصدار البروتوكول

- ثابت: `PROTOCOL_VERSION: u32 = 1` (في `scone-protocol`)؛
- يحمله `Hello.version` (المصافحة) و`BlockHeader.version`؛
- قواعد التوافق:
  - الإصدار المستلَم **أعلى** من الإصدار المحلي ← رفض
    (`ProtocolError::UnsupportedVersion`)؛ العقدة لا تخمّن
    صيغة إصدار لا تعرفه؛
  - الإصدار `0` ← غير صالح؛
  - توجد مجموعة صيغ واحدة فقط: بايتات إصدار المعاملة
    (`0x01`) وإصدار الكتلة يجب أن تطابق القيم المحلية
    **بالضبط**؛
- لا فصل major/minor في هذه المرحلة: عدد صحيح واحد، يُرفع
  عند أي كسر لصيغة الـ wire.

## مبادئ الصيغة الثنائية

- ثنائية، وليس JSON أبدًا لبيانات البروتوكول؛
- حتمية وقانونية: كائنان متساويان منطقيًا ينتجان **نفس
  البايتات بالضبط**؛
- تحليل تسلسلي، دون تخصيص ذاكرة حيثما أمكن (استعارات بايتات)؛
- كل مدخلات الشبكة غير موثوقة: تُفحص الأطوال والعدادات
  **قبل** أي تخصيص؛ فك الترميز لا يُسبب panic أبدًا.

### ترتيب البايتات (Endianness)

- جميع الأعداد الصحيحة varints بترميز LEB128 (مجموعات
  7-بت، الأدلى أولًا): لا ينطبق ترتيب بايتات الجهاز؛
- عناوين IPv4/IPv6 والمعرّفات 32-بايت: مصفوفات بايتات خام،
  بلا بادئة طول؛
- لا عدد صحيح عريض ثابت الحجم في الصيغة: قاعدة ترميز
  واحدة للأعداد الصحيحة.

### Varint (LEB128 الأدنى)

يحمل كل بايت 7 بتات من البيانات؛ البت الأعلى يساوي 1 في كل
بايت عدا الأخير. يسع `u64` في 1 إلى 10 بايت.

| القيمة | البايتات (سداسي عشري) |
|---|---|
| 0 | `00` |
| 1 | `01` |
| 127 | `7f` |
| 128 | `80 01` |
| 255 | `ff 01` |
| 16,384 | `80 80 01` |
| 2^32 | `80 80 80 80 10` |
| u64::MAX | `ff ff ff ff ff ff ff ff ff 01` |

يجب أن يكون الترميز **أدنى (minimal)**: الأشكال المتضخمة
تُرفض عند فك الترميز (`InvalidVarint`). هذا هو الشرط اللازم
لكي تكون لقيمة ما تسلسل بايتات صالح واحد فقط (ترميز
قانوني). حالات الرفض: `80 00` (غير أدنى)، `ff×9 02` (فيض
u64)، `ff×10` (طويل جدًا).

### السلاسل والبايتات

- `bytes`: طول varint + بايتات خام؛
- `str`: نفسه، مع فك ترميز UTF-8 إلزامي؛
- كل طول مسقوف (انظر [الحدود](#الحدود)) ويُفحص قبل
  القراءة.

### المعرّفات

| النوع | التمثيل على الـ wire |
|---|---|
| `DomainId` | 32 بايت خام |
| `TldId` | 32 بايت خام |
| `OwnerId` | 32 بايت خام |
| `PublicKeyRef` | 32 بايت خام |
| `RecordHash` | 32 بايت خام |
| `BlockHash` | 32 بايت خام |
| `MerkleRoot` | 32 بايت خام |
| `DomainName` | `str` (≤ 253 بايت، يعاد التحقق منه في `scone-core`) |

تبقى الاشتقاقات (`BLAKE3-256` + بادئات `SCONE-DOMAIN-V1`
وغيرها) معرّفة في `scone-core` (انظر
`/docs/en/general/naming.md`).

## المعاملات

```text
Transaction = disc u8 || version u8 || payload
```

| الـ discriminant | النوع |
|---|---|
| `0x21` | `RegisterDomain` |
| `0x2A` | `RegisterTldPow` |
| `0x52` | `UpdateDomain` |
| `0x57` | `RenewTldPow` |
| `0x84` | `RegisterDomainPow` |
| `0x93` | `RegisterTld` |
| `0xC3` | `RenewDomainPow` |

كل discriminant ثابت معتم اعتباطي (لا تسلسل منطقي؛
`0x00` غير صالح أبدًا) — الجدول المعياري الكامل في
`/docs/en/technical/transactions.md`.

يتبع بايت `version` (القيمة الحالية `0x01`) الـ
discriminant: أي قيمة أخرى تُرفض صراحةً
(`UnsupportedVersion(value)`). أغلب المعاملات موقّعة
(Ed25519) — عدا صيغ pow-to-owner الأربع بلا توقيع
(`0x2A`، `0x57`، `0x84`، `0xC3`): الإثبات المُستخرَج بالتعدين هو
التخويل نفسه، والـ challenge الخاص به يربط مفتاح المستفيد
بالشبكة. المواصفة الكاملة، الحمولة الموقّعة وقواعد التحقق في
**`/docs/en/technical/transactions.md`** (معيارية).

منذ تغيير الـ wire بتاريخ 2026-09، **لم تعد المعاملات تحمل
حقل `network`**: يُحقن `network_id` (`str` ≤ 16) في **حمولة
التوقيع** (الصيغ الموقّعة) أو **challenge الـ PoW**
(صيغ pow-to-owner) — المعاملة المُعدّة لشبكة واحدة لا تنتقل
أبدًا إلى أخرى (انظر `/docs/en/technical/blockchain.md`).

### REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

تحمل `RegisterDomain` **الاسم القانوني بصيغة واضحة**، يليه
`DomainId` الخاص به — يُعاد فحص الاتساق `domain_id ==
DomainId(name)` عند فك الترميز (يُعاد حسابه، لا يُوثق أبدًا).
يحمل `proof` الـ PoW الخاص بالتسجيل (20 بايت: nonce + epoch
+ difficulty — ملح الـ epoch بقيمة `epoch = height/144` للكتلة
المطبِّقة، يُتحقق منه بصرامة عند التطبيق مقابل ثابت الشبكة
وepoch الكتلة؛ رفض مُصنَّف `ProofEpochMismatch` عند عدم
التطابق — M8b، انظر `transactions.md`). الـ `public_key` هو
مفتاح Ed25519 للموقّع (32 بايت بالضبط)؛ ويجب أن يكون
`owner` اشتقاق BLAKE3 لذلك المفتاح (يُعاد حسابه، لا يُوثق
أبدًا)؛ و`signature` 64 بايت بالضبط.

### REGISTER_TLD (0x93)

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

مطالبة بـ TLD (سجل منفصل عن سجل النطاقات، M7): يُحمَل
الـ TLD **بصيغة واضحة** كاسم النطاق (M8b — تحدي الـ PoW
يُشتق من الاسم)، و`tld_id` هو اشتقاق `SCONE-TLD-V1` لذلك
الاسم، منفصل عن كل `DomainId` بالبناء. قواعد الحالة (M8b):
TLD حر + غير محجوز لـ ICANN (`IcannTldReserved` — نطاقات
Icann الجذرية لا يطالب بها أحد) + PoW بصعوبة العلامة
**الحسب-طولية** (منحنى `pow_length` — انظر `blockchain.md`)؛
والمطالَب يكون مغلقًا (assign فقط).

### UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

ثابت يُعاد فحصه عند فك الترميز: `sequence > 0` و`owner` ==
اشتقاق `public_key`. لا `timestamp`: الترتيب يمر عبر
`sequence`. تحمل المعاملة مراجع مدمجة فقط — محتوى DNS
الكامل يبقى في الـ DHT (انظر أدناه).

## سجلات DNS (جانب الـ DHT)

سجلات DNS الكاملة لا تدخل البلوكشين أبدًا: تُتبادل عبر
الـ DHT (رسائل `GetRecord`/`Record`) وتُلتزم على السلسلة
بـ `RecordHash` فقط.

### RecordData

```text
RecordData = type.v || ttl.v || payload(type)
```

رموز الأنواع هي **رموز DNS الحقيقية** (IANA). يتبع الـ TTL
(ثوانٍ، varint) رمز النوع ويغطي السجل كله: الحدود
`MIN_TTL = 1` .. `MAX_TTL = 2,592,000` (30 يومًا؛ الافتراضي
`DEFAULT_TTL = 86,400`).

| الرمز | النوع | الحمولة |
|---|---|---|
| 1 | A | 4 بايت |
| 2 | NS | `name` |
| 5 | CNAME | `name` |
| 12 | PTR | `name` |
| 15 | MX | `preference.v`، `name` |
| 16 | TXT | `str` ≤ 4096 |
| 28 | AAAA | 16 بايت |
| 33 | SRV | `priority.v`، `weight.v`، `port.v`، `target(name)` |
| 257 | CAA | `flags u8`، `tag` (1–15 بايت)، `value` (`str` ≤ 4096) |
| 32768 | GEOROUTE | GeoRoute (انظر أدناه) |
| غيره | Unknown | code.v + `bytes` ≤ 4096 |

مثال: `A 192.0.2.1` (بTTL قدره 86,400) ←
`01 80 a3 05 c0 00 02 01`
(النوع، TTL بصيغة varint، العنوان).

يُرفض الـ `Unknown` الحامل لرمز معروف عند الترميز (فك
الترميز لا ينتجه أبدًا).

#### GEOROUTE (GeoDNS)

نوع خاص بـ Scone (نطاق غير IANA). قاعدة توجيه جغرافي
واحدة:

```text
GeoRoute  = scope || count.v(≤ 32) || endpoint…
scope     = 0x00                    -- عالمي
          | 0x01 continent.v        -- UN M49
          | 0x02 country.v          -- ISO 3166-1 رقمي
          | 0x03 region.v           -- رمز المنطقة
          | 0x04 asn.v              -- رقم AS
          | 0x05 prefix.addr[16] || prefix.len u8   -- بادئة IP (موحَّدة)
endpoint  = target || priority.v || weight.v
target    = 0x00 octets[4]          -- IPv4 ثنائي
          | 0x01 octets[16]         -- IPv6 ثنائي
```

**القاعدة القانونية**: تُرمَّز نقاط نهاية نطاق (scope) معيّن
**مرتّبة** (الأولوية تصاعديًا، ثم الوزن تنازليًا، ثم الهدف
تصاعديًا)؛ يُطبَّق الترتيب عند الترميز ويُرفض الـ wire غير
المرتب عند فك الترميز (`NonCanonical`)، كما في مجموعات
السجلات. التكرارات الكاملة لنقاط النهاية تُرفض.

**التحقق**: `priority > 0`، `weight ≤ 100` (0 = نقطة نهاية
غير نشطة)، ≤ 32 نقطة نهاية لكل نطاق (`GEO_MAX_ENDPOINTS`).
رمز بلد/قارة غير معروف للجدول المحلي
(`continent_of_country`) يُقبل عند مستوى الـ wire: قد يكون
جدول المحلّل أحدث من جدول الناشر. الـ TTL هو الخاص بالـ
`RecordData` الأب (دون تغيير).

### DnsRecord

```text
domain_id[32] sequence.v expiration.v count.v(≤ 256) records…
```

**القاعدة القانونية**: مجموعة السجلات تجميع غير مرتب. تُرمَّز
**مرتّبة بالترتيب المعجمي لترميز كل سجل فردي**، تصاعديًا
صرامةً (تُرفض التكرارات). يُرفض الـ wire غير المرتب عند فك
الترميز (`NonCanonical`). ترتيب الـ `Vec` الأصلي لا يُنقل.

### SignedDnsRecord

```text
DnsRecord owner[32] signature(bytes ≤ 128)
```

### RecordHash

```text
RecordHash = BLAKE3-256("SCONE-RECORD-V1" || canonical(DnsRecord))
```

يغطي الـ hash **محتوى السجل وحده**؛ ويُتحقق من توقيع المالك
منفصلًا فوق نفس البايتات القانونية. *(قرار: الالتزام لا يغطي
التوقيع — نسخة سابقة من هذه الوثيقة ذكرت "SignedDnsRecord
الكامل"؛ ربط الـ hash بالتوقيع لم يضف شيئًا وقيّد الالتزام
بنظام التوقيع.)*

## الكتل

### BlockHeader

```text
version.v height.v prev_hash[32] tx_root[32] timestamp.v consensus(bytes ≤ 65,536)
```

- `version`: إصدار البروتوكول (القواعد أعلاه)؛
- `height`: الارتفاع (التكوين = 0)؛
- `prev_hash`: hash الكتلة السابقة (أصفار لكتلة التكوين)؛
- `tx_root`: التزام فوق قائمة المعاملات (شجرة Merkle
  تعرّفها `scone-blockchain`، انظر
  `/docs/en/technical/blockchain.md`)؛
- `timestamp`: معلومة ترتيب (Unix، ثوانٍ)؛
- `consensus`: حمولة المنتج (`SCONE-BLOCK-V1 ‖ pk[32] ‖
  sig[64]`، انظر `blockchain.md`) — يحمل تكوينُ شبكةٍ الـ
  `network_id` الخام، وقد يحمل تكوين التخصيصات (mainnet
  kickstarter) manifest بقيمة `ALLOC-V1` (≤ 60 KiB)؛ محدود
  بـ `MAX_CONSENSUS_LEN` (64 KiB).

### Block

```text
BlockHeader tx_count.v(≤ 4096) transactions…
```

تحليل تسلسلي؛ يُسقف العداد **قبل** التخصيص؛ وأي اقتطاع
يُرفض. بخلاف مجموعات السجلات، ترتيب المعاملات **معنوي**
(ترتيب التوافق) ويُحفظ بالضبط.

يُحسب `BlockHash` (hash الترويسة القانونية) و`tx_root` في
`scone-blockchain` فوق الترميزات المعرّفة هنا (الصيغ الدقيقة
في `/docs/en/technical/blockchain.md`).

## رسائل P2P

```text
Message = disc u8 || payload
```

النقل (التأطير، libp2p…) خارج النطاق؛ ينبغي ألا يتجاوز إطار
نقل واحد `MAX_MESSAGE_LEN` (1 MiB).

| الـ disc | الرسالة | الاتجاه | الحمولة | الحد | الرد المتوقع |
|---|---|---|---|---|---|
| `0x01` | `Hello` | كلاهما | `version.v` | — | `Hello` |
| `0x02` | `Ping` | كلاهما | — | — | `Pong` |
| `0x03` | `Pong` | كلاهما | — | — | — |
| `0x04` | `GetBlock` | كلاهما | `hash[32]` | — | `Block` |
| `0x05` | `GetBlocks` | كلاهما | `start.v` `max.v` | `max ≤ 128` | ≤ `max` × `Block` |
| `0x06` | `Block` | كلاهما | `Block` | tx ≤ 4096 | — |
| `0x07` | `Transaction` | كلاهما | `Transaction` | — | — |
| `0x08` | `GetRecord` | كلاهما | `domain_id[32]` | — | `Record` |
| `0x09` | `Record` | كلاهما | `SignedDnsRecord` | — | — |
| `0x0a` | `Checkpoint` | كلاهما | `Checkpoint` | موقّعون ≤ 64 | — |
| `0x0b` | `GetCheckpoints` | كلاهما | — | — | نافذة نقاط التفتيش |
| `0x0c` | `GetStatus` | كلاهما | — | — | `NodeStatus` |
| `0x0d` | `NodeStatus` | كلاهما | حالة العقدة (الشبكة، القمة، الارتفاع، NAT، النطاقات، mempool، الأقران، النهائية، المشغّلون الطازجون) | — | — |
| `0x0e` | `LookupName` | كلاهما | `name(str)` | — | `NameState` |
| `0x0f` | `NameState` | كلاهما | الحالة on-chain للاسم + الـ TLD | — | — |
| `0x10` | `GetOwnedNames` | كلاهما | `owner[32]` | — | `OwnedNames` |
| `0x11` | `OwnedNames` | كلاهما | النطاقات الحية للمالك + الـ TLD | — | — |
| `0x12` | `Error` | رد | رسالة خطأ على مستوى التطبيق (`str`) | — | — |

### نقطة التفتيش Checkpoint (M5)

تذهب نقطة التفتيش المجمّعة إلى الـ wire كالتالي: وسم
`SCONE-CKPT-V1` || جسم `CheckpointData` (116 بايت، LE — مطابق
لنهاية preimage التوقيع: `epoch(8) height(8) block_hash(32)
prev_checkpoint_hash(32) state_root(32) recovery(4)`) ||
`count.v` || `pk[32] ‖ sig[64]` لكل موقّع، **مرتّبة بـ pk
تصاعديًا، بلا تكرارات**، `count ≤ 64`
(`MAX_CHECKPOINT_SIGNERS`). فك ترميز صارم: الاقتطاع، الوسم
الخاطئ، pk غير قانوني، التكرارات أو عدم الترتيب = رفض
(`None`)، وليس panic أبدًا (تلف fuzz مُختبَر).

يطلب `GetCheckpoints` نافذة النهائية؛ والرد هو نافذة
`checkpoint_window()` (على الأكثر `CHECKPOINT_KEEP` نقطة
تفتيش، من الأقدم إلى الأحدث) عبر رسالة `Checkpoint` واحدة
أو أكثر.

أصبح `TxId` معرّفًا الآن في `scone-blockchain`
(`BLAKE3-256("SCONE-TX-V1" || canonical(Transaction))`، انظر
`/docs/en/technical/blockchain.md`). الـ mempool ومنع إعادة
التشغيل المُنوَّع (`TxReplay`، `REPLAY_WINDOW_BLOCKS = 256`)
مُسلَّمان في المُرحّل (M4)؛ ولا يبقى مؤجلًا إلا
`GetTransaction` (تعريفه يتوقف على قرارات التوافق
المتبقية).

## فصل البلوكشين / الـ DHT

```text
BLOCKCHAIN                      DHT
──────────                      ───
domain_id                       domain_id
owner                           SignedDnsRecord (كامل، موقّع)
sequence
record_hash
```

تقول السلسلة *من* يملك و*ما الحالة الراهنة* (sequence +
record_hash). ويوفّر الـ DHT *المحتوى*. يُقبل سجل الـ DHT
إذا وفقط إذا:

1. كان توقيعه صالحًا للمالك المسجّل على السلسلة؛
2. كان `sequence` الخاص به متسقًا مع (أو أعلى من) الخاص
   بالسلسلة؛
3. طابق `RecordHash` المعاد حسابه فوق ترميزه القانوني الـ
   `record_hash` الخاص بالسلسلة.

## الحدود

| الثابت | القيمة | الدور |
|---|---|---|
| `MAX_NAME_LEN` | 253 | طول الاسم |
| `MAX_TLD_LEN` | 63 | طول الـ TLD |
| `MAX_TXT_LEN` | 4096 | سلسلة TXT |
| `MAX_PROOF_LEN` | 256 | إثبات التسجيل |
| `MAX_SIGNATURE_LEN` | 128 | التوقيع (Ed25519 = 64) |
| `MAX_RECORDS_PER_SET` | 256 | سجلات لكل مجموعة |
| `MAX_DHT_RECORD_LEN` | 64 KiB | سجل DNS الموقّع المرمَّز (حمولة DHT: wire، تخزين، gossip) — حدّ إجمالي يُفحص عند الترميز وقبل أي فك ترميز؛ أعلى بـ 15–30× من أي مجموعة واقعية (SPF+DKIM+DMARC ≈ 2–5 KiB)، وأدنى بـ 16× من سقف النقل. |
| `GEO_MAX_ENDPOINTS` | 32 | نقاط النهاية لكل نطاق GeoRoute |
| `MAX_UNKNOWN_DATA` | 4096 | بيانات نوع غير معروف |
| `MAX_TXS_PER_BLOCK` | 4096 | معاملات لكل كتلة |
| `MAX_CONSENSUS_LEN` | 64 KiB | حمولة التوافق في ترويسة (وُسِّع لـ manifest التكوين `ALLOC-V1` — mainnet kickstarter) |
| `MAX_BLOCKS_PER_REQUEST` | 128 | كتل لكل `GetBlocks` |
| `MAX_MESSAGE_LEN` | 1 MiB | إطار النقل (موصى به) |

أي تجاوز ← `LimitExceeded`، يُكتشف قبل التخصيص: لا يستطيع
قرين خبيث التسبب بتخصيص غير متناسب عند فك الترميز.

## الترميز القانوني

القاعدة الأساسية:

```text
كائن قانوني ← ترميز ثنائي قانوني ← hash / توقيع
```

- varints أدنى فقط؛
- مجموعات السجلات مرتّبة تصاعديًا صرامةً؛
- للكائن المتساوي منطقيًا ترميز صالح **واحد** بالضبط؛
- يرفض فك الترميز الأشكال غير القانونية (`NonCanonical`)،
  فلا توجد ترميزات صالحة متعددة لنفس الكائن.

## معالجة الأخطاء

`ProtocolError` (في `scone-protocol::error`):

| الـ variant | المعنى |
|---|---|
| `Truncated` | انتهى المُدخل في منتصف قيمة |
| `InvalidVarint` | varint غير أدنى أو فيض أو طويل جدًا |
| `IntegerOutOfRange` | عدد صحيح لا يسع نوعه الهدف |
| `UnsupportedVersion` | إصدار صفر أو أعلى من المحلي |
| `UnknownDiscriminant` | discriminant معاملة/رسالة غير معروف |
| `LimitExceeded` | طول أو عدد يتجاوز حدًا |
| `InvalidUtf8` | سلسلة غير UTF-8 |
| `TrailingBytes` | بايتات متبقية بعد قيمة كاملة |
| `Validation` | خرق ثابت من ثوابت `scone-core` |
| `NonCanonical` | ترميز صالح لكل حقل لكن غير قانوني |

فك الترميز لا يُسبب panic أبدًا على بيانات الشبكة.

## التوقيعات

- النظام: **Ed25519** (RFC 8032) عبر `scone-crypto`
  (`ed25519-dalek`، تحقق `verify_strict` — التوقيعات
  القابلة للتشكيل والمفاتيح الضعيفة تُرفض)؛
- **معاملات السلسلة**: كل `RegisterDomain` / `RegisterTld` /
  `UpdateDomain` تضمّن `public_key` (32 بايت) + `signature`
  (64 بايت) فوق الحمولة
  `"SCONE-TX-SIG-V1" || canonical_encode(tx_without_signature)`؛
  المواصفة المعيارية الكاملة: `/docs/en/technical/transactions.md`؛
- **سجلات الـ DHT** (`SignedDnsRecord`): الحمولة الموقّعة هي
  الترميز القانوني للـ `DnsRecord`؛ التحقق يقاطع
  `record + owner + signature` مع الهوية المسجّلة في
  البلوكشين؛
- قاعدة عرضية: يُعاد حساب `owner` وجميع الـ
  hashes/الالتزامات، ولا تُؤخذ من قيمة مقدَّمة أبدًا.

## قواعد التوافق

مُسلَّمة (موصوفة في `/docs/en/technical/blockchain.md`): منتج
الكتل الموقّع (حمولة `SCONE-BLOCK-V1`، لجنة منتخبة بنقاط
التفتيش)، النهائية بنقاط تفتيش مجمّعة (`SCONE-CKPT-V1`،
اختيار الفرع عند أرضية النهائية)، PoW التسجيل حسب طول
الاسم. المواضيع المتبقية: رسوم مضادة للسبام دقيقة، تدوير
مفتاح المالك.

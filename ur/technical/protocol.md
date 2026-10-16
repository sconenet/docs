# پروٹوکول

Scone پروٹوکول کے لیے حوالہ دستاویز: بائنری فارمیٹ، کیننیکل
انکوڈنگ، ٹرانزیکشنز، DNS ریکارڈز، بلاکس، P2P پیغامات، حدود اور
ورژن ہم آہنگی۔ حوالہ نفاذ
[`scone-protocol`](../../../crates/scone-protocol) crate ہے۔

**(عارضی)** نشان زد اشیاء حتمی نہیں ہیں۔

## پروٹوکول ورژن

- مستقل: `PROTOCOL_VERSION: u32 = 1` (`scone-protocol`)؛
- `Hello.version` (ہینڈ شیک) اور `BlockHeader.version` کے ذریعے
  لے جایا جاتا ہے؛
- ہم آہنگی کے قواعد:
  - موصول ورژن مقامی ورژن سے **زیادہ** → مسترد
    (`ProtocolError::UnsupportedVersion`)؛ نوڈ اس ورژن کا فارمیٹ
    نہیں اندازہ کرتا جسے وہ نہیں جانتا؛
  - ورژن `0` → غلط؛
  - فارمیٹس کا صرف ایک سیٹ ہے: ٹرانزیکشن ورژن بائٹس
    (`0x01`) اور بلاک ورژن مقامی قیمتوں سے **بالکل** ملتے
    ہوں گے؛
- اس مرحلے پر کوئی major/minor تقسیم نہیں: ایک ہی عدد، جو
  وائر فارمیٹ کے ہر وقفے پر بڑھایا جاتا ہے۔

## بائنری فارمیٹ کے اصول

- بائنری، پروٹوکول ڈیٹا کے لیے کبھی JSON نہیں؛
- متعین اور کیننیکل: دو منطقی برابر اشیاء **بالکل وہی بائٹس**
  بناتی ہیں؛
- ترتیبی پارسنگ، جہاں ممکن ہو بغیر مختص (بائٹ borrows)؛
- تمام نیٹ ورک ان پٹ غیر بھروسہ مند: لمبائیاں اور گنتی کسی بھی
  مختص سے **پہلے** چیک ہوتی ہیں؛ ڈی کوڈنگ کبھی panic نہیں کرتی۔

### Endianness

- تمام اعداد LEB128 varints ہیں (7 بٹ گروہس، سب سے کم اہم
  پہلے): مقامی endianness لاگو نہیں ہوتی؛
- IPv4/IPv6 پتے اور 32 بائٹ شناخت کنندگان: خام بائٹ قطاریں، کوئی
  لمبائی سابقہ نہیں؛
- فارمیٹ میں کوئی مقررہ سائز کا چوڑا عدد نہیں: ایک ہی عدد
  انکوڈنگ قاعدہ۔

### Varint (کم سے کم LEB128)

ہر بائٹ 7 بٹ ڈیٹا لیتی ہے؛ آخری کے سوا ہر بائٹ پر ہائی بٹ 1
ہے۔ ایک `u64` 1 سے 10 بائٹس میں سمٹتا ہے۔

| قیمت | بائٹس (ہیکس) |
|---|---|
| 0 | `00` |
| 1 | `01` |
| 127 | `7f` |
| 128 | `80 01` |
| 255 | `ff 01` |
| 16,384 | `80 80 01` |
| 2^32 | `80 80 80 80 10` |
| u64::MAX | `ff ff ff ff ff ff ff ff ff 01` |

انکوڈنگ **کم سے کم** ہونی چاہیے: بڑی شکلیں ڈی کوڈنگ پر
مسترد (`InvalidVarint`)۔ یہ اس شرط کی ضرورت ہے کہ ہر قیمت کی
صرف ایک درست بائٹ ترتیب ہو (کیننیکل انکوڈنگ)۔ انکاری:
`80 00` (غیر کم سے کم)، `ff×9 02` (u64 اوور فلو)، `ff×10`
(بہت لمبی)۔

## اسٹرنگز اور بائٹس

- `bytes`: varint لمبائی + خام بائٹس؛
- `str`: وہی، لازمی UTF-8 ڈی کوڈنگ کے ساتھ؛
- ہر لمبائی capped ہے ([حدود](#limits) دیکھیں) اور پڑھنے سے
  پہلے چیک ہوتی ہے۔

### شناخت کنندگان

| نوعیت | وائر نمائندگی |
|---|---|
| `DomainId` | 32 خام بائٹس |
| `TldId` | 32 خام بائٹس |
| `OwnerId` | 32 خام بائٹس |
| `PublicKeyRef` | 32 خام بائٹس |
| `RecordHash` | 32 خام بائٹس |
| `BlockHash` | 32 خام بائٹس |
| `MerkleRoot` | 32 خام بائٹس |
| `DomainName` | `str` (≤ 253 بائٹس، `scone-core` سے دوبارہ تصدیق) |

اخذ (`BLAKE3-256` + `SCONE-DOMAIN-V1` سابقے، وغیرہ) `scone-core`
کی طرف سے متعین رہتے ہیں (دیکھیں `/docs/en/general/naming.md`)۔

## ٹرانزیکشنز

```text
Transaction = disc u8 || version u8 || payload
```

| ڈسکریمیننٹ | نوعیت |
|---|---|
| `0x21` | `RegisterDomain` |
| `0x52` | `UpdateDomain` |
| `0x93` | `RegisterTld` |

ہر ڈسکریمیننٹ ایک غیر متعین اپاہج مستقل ہے (کوئی منطقی
تسلسل نہیں؛ `0x00` کبھی درست نہیں) — مکمل نافذہ جدول
`/docs/en/technical/transactions.md` میں۔

`version` بائٹ (موجودہ قیمت `0x01`) ڈسکریمیننٹ کے پیچھے آتی ہے:
کوئی اور قیمت واضح طور پر مسترد
(`UnsupportedVersion(value)`)۔ **تمام ٹرانزیکشنز دستخط شدہ
ہیں** (Ed25519): مکمل تفصیل، دستخط شدہ پے لوڈ اور تصدیق کے قواعد
**`/docs/en/technical/transactions.md`** (نافذہ) میں۔

2026-09 وائر تبدیلی سے، **ٹرانزیکشنز میں اب کوئی `network`
فیلڈ نہیں**: `network_id` (`str` ≤ 16) صرف **دستخطی پے لوڈ**
میں ڈالا جاتا ہے — ایک نیٹ ورک کے لیے دستخط شدہ tx کبھی
دوسرے پر تصدیق نہیں ہوتی (دیکھیں `/docs/en/technical/blockchain.md`)۔

### REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

`RegisterDomain` **کیننیکل نام صاف الفاظ میں** رکھتا ہے، اس کے
بعد اس کی `DomainId` — مطابقت `domain_id == DomainId(name)` ڈی
کوڈنگ پر دوبارہ چیک ہوتی ہے (دوبارہ حساب، کبھی بھروسہ نہیں)۔
`proof` رجسٹریشن PoW رکھتا ہے (20 بائٹس: nonce + epoch + مشکل —
epoch نمک `epoch = height/144` اطلاق والے بلاک کا، اطلاق پر
نیٹ ورک مستقل اور بلاک کے epoch کے خلاف سخت تصدیق؛ بے میلانی پر
ٹائپ شدہ انکار `ProofEpochMismatch` — M8b، دیکھیں
`transactions.md`)۔ `public_key` دستخط کنندہ کی Ed25519 کلید ہے
(بالکل 32 بائٹس)؛ `owner` اُس کلید کا BLAKE3 حصول ہونا چاہیے
(دوبارہ حساب، کبھی بھروسہ نہیں)؛ `signature` بالکل 64 بائٹس ہے۔

### REGISTER_TLD (0x93)

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

TLD کا دعوی (رجسٹری، ڈومین رجسٹری سے الگ، M7):
TLD ڈومین کے نام کی طرح **صاف الفاظ میں** لے جایا جاتا ہے
(M8b — PoW چیلنج نام سے اخذ ہوتا ہے)، اور `tld_id` اُس نام کا
`SCONE-TLD-V1` حصول ہے، تعمیر ہی سے ہر `DomainId` سے الگ۔ اسٹیٹ
قواعد (M8b): TLD آزاد + ICANN-محفوظ نہیں
(`IcannTldReserved` — ICANN روٹ TLDs کا دعوی کوئی نہیں کر سکتا)
+ لیبل کی **لمبائی پر مبنی** مشکل پر PoW (`pow_length` منحنی —
دیکھیں `blockchain.md`)؛ دعوی شدہ بند ہے (صرف assign)۔

### UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

ڈی کوڈنگ پر دوبارہ چیک شدہ مستقل حقیقت: `sequence > 0` اور
`owner` == `public_key` کا حصول۔ کوئی `timestamp` نہیں: ترتیب
`sequence` سے ہوتی ہے۔ ٹرانزیکشن صرف کمپیکٹ حوالے رکھتی ہے — مکمل
DNS مواد DHT میں رہتا ہے (نیچے دیکھیں)۔

## DNS ریکارڈز (DHT کی جانب)

مکمل DNS ریکارڈز کبھی بلاکچین میں داخل نہیں ہوتے: یہ DHT
(`GetRecord`/`Record` پیغامات) کے ذریعے تبادلہ ہوتے ہیں اور
چین پر صرف اپنے `RecordHash` سے commit ہوتے ہیں۔

### RecordData

```text
RecordData = type.v || ttl.v || payload(type)
```

ٹائپ کوڈز **اصل DNS کوڈز** ہیں (IANA)۔ TTL (سیکنڈ،
varint) ٹائپ کوڈ کے بعد آتا ہے اور پورے ریکارڈ کو ڈھانکتا ہے:
حدودیں `MIN_TTL = 1` .. `MAX_TTL = 2,592,000` (30 دن؛ ڈیفالٹ
`DEFAULT_TTL = 86,400`)۔

| کوڈ | نوعیت | پے لوڈ |
|---|---|---|
| 1 | A | 4 بائٹس |
| 2 | NS | `name` |
| 5 | CNAME | `name` |
| 12 | PTR | `name` |
| 15 | MX | `preference.v`، `name` |
| 16 | TXT | `str` ≤ 4096 |
| 28 | AAAA | 16 بائٹس |
| 33 | SRV | `priority.v`، `weight.v`، `port.v`، `target(name)` |
| 257 | CAA | `flags u8`، `tag` (1–15 B)، `value` (`str` ≤ 4096) |
| 32768 | GEOROUTE | GeoRoute (نیچے دیکھیں) |
| دیگر | Unknown | code.v + `bytes` ≤ 4096 |

مثال: `A 192.0.2.1` (TTL 86,400) → `01 80 a3 05 c0 00 02 01`
(ٹائپ، TTL varint، پتہ)۔

ایک `Unknown` جو معروف کوڈ رکھتا ہو انکوڈنگ پر مسترد ہے (ڈی
کوڈنگ کبھی ایسا نہیں بنا سکتی)۔

#### GEOROUTE (GeoDNS)

ذاتی Scone نوعیت (غیر IANA حد)۔ ایک جغرافیائی روٹنگ قاعدہ:

```text
GeoRoute  = scope || count.v(≤ 32) || endpoint…
scope     = 0x00                    -- world
          | 0x01 continent.v        -- UN M49
          | 0x02 country.v          -- ISO 3166-1 numeric
          | 0x03 region.v           -- region code
          | 0x04 asn.v              -- AS number
          | 0x05 prefix.addr[16] || prefix.len u8   -- IP prefix (normalized)
endpoint  = target || priority.v || weight.v
target    = 0x00 octets[4]          -- binary IPv4
          | 0x01 octets[16]         -- binary IPv6
```

**کیننیکل قاعدہ**: کسی دیے گئے scope کے endpoints **ترتیب شدہ**
انکوڈ ہوتے ہیں (priority بڑھتی، پھر weight گھٹتی، پھر target
بڑھتی)؛ ترتیب انکوڈنگ پر لاگو ہوتی ہے اور غیر ترتیب شدہ وائر ڈی
کوڈنگ پر مسترد (`NonCanonical`)، ریکارڈ سیٹس کی طرح۔ مکمل
endpoint ڈپلیکیٹس مسترد۔

**تصدیق**: `priority > 0`، `weight ≤ 100` (0 = غیر فعال
endpoint)، فی scope ≤ 32 endpoints (`GEO_MAX_ENDPOINTS`)۔ کوئی
ملک/براعظم کوڈ جو مقامی جدول (`continent_of_country`) سے نامعلوم
ہو وائر سطح پر **قبول** ہے: ریزالور کا جدول اشاعتی کار سے زیادہ
تازہ ہو سکتا ہے۔ TTL والد `RecordData` کا ہے (بے تبدیل)۔

### DnsRecord

```text
domain_id[32] sequence.v expiration.v count.v(≤ 256) records…
```

**کیننیکل قاعدہ**: ریکارڈ سیٹ ایک بے ترتیب مجموعہ ہے۔ یہ **ہر
ریکارڈ کی انفرادی انکوڈنگ کے لغوی ترتیب سے ترتیب شدہ** انکوڈ ہوتا
ہے، سخت بڑھتی (ڈپلیکیٹس مسترد)۔ غیر ترتیب شدہ وائر ڈی کوڈنگ پر
مسترد (`NonCanonical`)۔ اصل `Vec` کی ترتیب لے جایا نہیں جاتی۔

### SignedDnsRecord

```text
DnsRecord owner[32] signature(bytes ≤ 128)
```

### RecordHash

```text
RecordHash = BLAKE3-256("SCONE-RECORD-V1" || canonical(DnsRecord))
```

ہیش صرف **ریکارڈ کے مواد** کو ڈھانکتا ہے؛ مالک کا دستخط اُنہی
کیننیکل بائٹس پر الگ تصدیق ہوتا ہے۔ *(فیصلہ: commitment دستخط کو
نہیں ڈھانکتا — اس دستاویز کے ایک پرانے ورژن میں "مکمل
SignedDnsRecord" لکھا تھا؛ ہیش کو دستخط سے جوڑنا کچھ حاصل نہیں
کرتا تھا اور commitment کو دستخط سکیم سے باندھ دیتا تھا۔)*

## بلاکس

### BlockHeader

```text
version.v height.v prev_hash[32] tx_root[32] timestamp.v consensus(bytes ≤ 65,536)
```

- `version`: پروٹوکول ورژن (اوپر کے قواعد)؛
- `height`: اونچائی (جنسیس = 0)؛
- `prev_hash`: پچھلے بلاک کا ہیش (جنسیس بلاک کے لیے صفر)؛
- `tx_root`: ٹرانزیکشن فہرست پر commitment (Merkle درخت،
  `scone-blockchain` سے متعین، دیکھیں `/docs/en/technical/blockchain.md`)؛
- `timestamp`: ترتیب کی معلومات (Unix، سیکنڈ)؛
- `consensus`: پیدا کار کا پے لوڈ (`SCONE-BLOCK-V1 ‖ pk[32] ‖
  sig[64]`، دیکھیں `blockchain.md`) — کسی نیٹ ورک کی جنسیس خام
  `network_id` رکھتی ہے، اور ایک الاٹمنٹ جنسیس (مین نیٹ
  kickstarter) ایک `ALLOC-V1` مینی فیسٹ (≤ 60 KiB) رکھ سکتی ہے؛
  `MAX_CONSENSUS_LEN` (64 KiB) سے محدود۔

### Block

```text
BlockHeader tx_count.v(≤ 4096) transactions…
```

ترتیبی پارسنگ؛ گنٹی مختص سے **پہلے** capped ہے؛ کوئی بھی
کتراؤ مسترد۔ ریکارڈ سیٹس کے برعکس، ٹرانزیکشن ترتیب **اہم** ہے
(کنسنسس ترتیب) اور بالکل محفوظ رہتی ہے۔

`BlockHash` (کیننیکل ہیڈر کا ہیش) اور `tx_root` `scone-blockchain`
یہاں متعین انکوڈنگز پر حساب کرتا ہے (درست فارمولے
`/docs/en/technical/blockchain.md` میں)۔

## P2P پیغامات

```text
Message = disc u8 || payload
```

ٹرانسپورٹ (فریمنگ، libp2p…) دائرۂ کار سے باہر؛ ایک ٹرانسپورٹ
فریم `MAX_MESSAGE_LEN` (1 MiB) سے بڑی نہ ہونی چاہیے۔

| Disc | پیغام | سمت | پے لوڈ | حد | متوقع جواب |
|---|---|---|---|---|---|
| `0x01` | `Hello` | دونوں | `version.v` | — | `Hello` |
| `0x02` | `Ping` | دونوں | — | — | `Pong` |
| `0x03` | `Pong` | دونوں | — | — | — |
| `0x04` | `GetBlock` | دونوں | `hash[32]` | — | `Block` |
| `0x05` | `GetBlocks` | دونوں | `start.v` `max.v` | `max ≤ 128` | ≤ `max` × `Block` |
| `0x06` | `Block` | دونوں | `Block` | tx ≤ 4096 | — |
| `0x07` | `Transaction` | دونوں | `Transaction` | — | — |
| `0x08` | `GetRecord` | دونوں | `domain_id[32]` | — | `Record` |
| `0x09` | `Record` | دونوں | `SignedDnsRecord` | — | — |
| `0x0a` | `Checkpoint` | دونوں | `Checkpoint` | signers ≤ 64 | — |
| `0x0b` | `GetCheckpoints` | دونوں | — | — | چیک پوائنٹ کھڑکی |
| `0x0c` | `GetStatus` | دونوں | — | — | `NodeStatus` |
| `0x0d` | `NodeStatus` | دونوں | نوڈ اسٹیٹ (نیٹ ورک، ٹِپ، اونچائی، NAT، ڈومینز، mempool، پیرز، فائنلٹی، تازہ آپریٹرز) | — | — |
| `0x0e` | `LookupName` | دونوں | `name(str)` | — | `NameState` |
| `0x0f` | `NameState` | دونوں | نام + TLD کی آن چین اسٹیٹ | — | — |
| `0x10` | `GetOwnedNames` | دونوں | `owner[32]` | — | `OwnedNames` |
| `0x11` | `OwnedNames` | دونوں | مالک کی زندہ ڈومینز + TLDs | — | — |
| `0x12` | `Error` | جواب | ایپلیکیشن سطح کا خرابی پیغام (`str`) | — | — |

### Checkpoint (M5)

ایک مجموعی چیک پوائنٹ وائر پر اس طرح جاتی ہے: ٹیگ `SCONE-CKPT-V1` ||
جسم `CheckpointData` (116 B، LE — دستخطی پری امیج کے آخر جیسا:
`epoch(8) height(8) block_hash(32) prev_checkpoint_hash(32)
state_root(32) recovery(4)`) || `count.v` || فی دستخط کنندہ
`pk[32] ‖ sig[64]`، **بڑھتی pk سے ترتیب شدہ، کوئی ڈپلیکیٹ نہیں**،
`count ≤ 64` (`MAX_CHECKPOINT_SIGNERS`)۔ سخت ڈی کوڈنگ: کتراؤ، غلط
ٹیگ، غیر کیننیکل pk، ڈپلیکیٹس یا غیر ترتیب شدہ = انکار (`None`)،
کبھی panic نہیں (خرابی فز ٹیسٹ شدہ)۔

`GetCheckpoints` فائنلٹی کھڑکی مانگتا ہے؛ جواب `checkpoint_window()`
کھڑکی ہے (زیادہ سے زیادہ `CHECKPOINT_KEEP` چیک پوائنٹس، پرانے سے
نئے) ایک یا زیادہ `Checkpoint` پیغامات کے ذریعے۔

`TxId` اب `scone-blockchain` کے ذریعے متعین ہے
(`BLAKE3-256("SCONE-TX-V1" || canonical(Transaction))`، دیکھیں
`/docs/en/technical/blockchain.md`)۔ Mempool اور کھڑکی دار
اینٹی ری پلے (`TxReplay`، `REPLAY_WINDOW_BLOCKS = 256`) ریلے میں
شپ شدہ (M4)؛ صرف `GetTransaction` ملتوی رہتا ہے (اس کی
تعریف باقی کنسنسس فیصلوں پر منحصر ہے)۔

## بلاکچین / DHT تقسیم

```text
BLOCKCHAIN                      DHT
──────────                      ───
domain_id                       domain_id
owner                           SignedDnsRecord (full, signed)
sequence
record_hash
```

چین کہتی ہے *کون* مالک ہے اور *موجودہ اسٹیٹ کیا ہے* (sequence +
record_hash)۔ DHT *مواد* فراہم کرتا ہے۔ ایک DHT ریکارڈ صرف اور
صرف اس صورت میں قبول ہوتا ہے:

1. اس کا دستخط چین پر رجسٹرڈ مالک کے لیے درست ہو؛
2. اس کا `sequence` چین کے ساتھ مطابق (یا اس سے بڑھا) ہو؛
3. اس کی کیننیکل انکوڈنگ پر دوبارہ حساب شدہ `RecordHash` چین کے
   `record_hash` سے ملتا ہو۔

## حدود

| مستقل | قیمت | کردار |
|---|---|---|
| `MAX_NAME_LEN` | 253 | نام کی لمبائی |
| `MAX_TLD_LEN` | 63 | TLD کی لمبائی |
| `MAX_TXT_LEN` | 4096 | TXT اسٹرنگ |
| `MAX_PROOF_LEN` | 256 | رجسٹریشن ثبوت |
| `MAX_SIGNATURE_LEN` | 128 | دستخط (Ed25519 = 64) |
| `MAX_RECORDS_PER_SET` | 256 | فی سیٹ ریکارڈز |
| `MAX_DHT_RECORD_LEN` | 64 KiB | انکوڈ شدہ دستخط شدہ DNS ریکارڈ (DHT پے لوڈ: وائر، اسٹوریج، گوسپ) — کل حد انکوڈنگ پر اور ہر ڈی کوڈنگ سے پہلے چیک؛ کسی بھی حقیقی سیٹ سے 15–30 گنا اوپر (SPF+DKIM+DMARC ≈ 2–5 KiB)، ٹرانسپورٹ کیپ سے 16 گنا نیچے۔ |
| `GEO_MAX_ENDPOINTS` | 32 | فی GeoRoute scope کے endpoints |
| `MAX_UNKNOWN_DATA` | 4096 | نامعلوم نوعیت کا ڈیٹا |
| `MAX_TXS_PER_BLOCK` | 4096 | فی بلاک ٹرانزیکشنز |
| `MAX_CONSENSUS_LEN` | 64 KiB | ہیڈر کا کنسنسس پے لوڈ (`ALLOC-V1` جنسیس مینی فیسٹ کے لیے چوڑا — مین نیٹ kickstarter) |
| `MAX_BLOCKS_PER_REQUEST` | 128 | فی `GetBlocks` بلاکس |
| `MAX_MESSAGE_LEN` | 1 MiB | ٹرانسپورٹ فریم (تجویز شدہ) |

کوئی بھی زیادتی → `LimitExceeded`، مختص سے پہلے پکڑی جاتی ہے:
کوئی معاندانہ پیر ڈی کوڈنگ پر غیر متناسب مختص نہیں کروا سکتا۔

## کیننیکل انکوڈنگ

بنیادی قاعدہ:

```text
canonical object → canonical binary encoding → hash / signature
```

- صرف کم سے کم varints؛
- ریکارڈ سیٹس سخت بڑھتی ترتیب میں؛
- ایک منطقی برابر شے کی بالکل **ایک** درست انکوڈنگ ہے؛
- ڈی کوڈنگ غیر کیننیکل شکلیں مسترد کرتی ہے (`NonCanonical`)،
  اس لیے ایک ہی شے کی متعدد درست انکوڈنگز نہیں ہیں۔

## خرابی کا انتظام

`ProtocolError` (`scone-protocol::error`):

| ویرئنٹ | معنی |
|---|---|
| `Truncated` | ان پٹ کسی قیمت کے بیچ میں ختم ہوا |
| `InvalidVarint` | غیر کم سے کم، اوور فلو یا بہت لمبا varint |
| `IntegerOutOfRange` | عدد اپنی ہدف نوعیت میں نہیں سمٹا |
| `UnsupportedVersion` | صفر یا مقامی سے بڑا ورژن |
| `UnknownDiscriminant` | نامعلوم ٹرانزیکشن/پیغام ڈسکریمیننٹ |
| `LimitExceeded` | حد سے باہر لمبائی یا گنتی |
| `InvalidUtf8` | غیر UTF-8 اسٹرنگ |
| `TrailingBytes` | مکمل قیمت کے بعد باقی بائٹس |
| `Validation` | کسی `scone-core` مستقل حقیقت کی خلاف ورزی |
| `NonCanonical` | فی فیلڈ درست مگر غیر کیننیکل انکوڈنگ |

ڈی کوڈنگ نیٹ ورک ڈیٹا پر کبھی panic نہیں کرتی۔

## دستخط

- سکیم: **Ed25519** (RFC 8032) `scone-crypto` کے ذریعے
  (`ed25519-dalek`، `verify_strict` تصدیق — لچکدار
  دستخط اور کمزور کلیدیں مسترد)؛
- **چین ٹرانزیکشنز**: ہر `RegisterDomain` / `RegisterTld` /
  `UpdateDomain` میں `public_key` (32 B) + `signature` (64 B)
  پے لوڈ `"SCONE-TX-SIG-V1" || canonical_encode(tx_without_signature)`
  پر؛ مکمل نافذہ تفصیل: `/docs/en/technical/transactions.md`؛
- **DHT ریکارڈز** (`SignedDnsRecord`): دستخط شدہ پے لوڈ
  `DnsRecord` کی کیننیکل انکوڈنگ ہے؛ تصدیق `record + owner +
  signature` کو بلاکچین میں رجسٹرڈ شناخت کے ساتھ کراس کرتی ہے؛
- پار جاتے قاعدہ: `owner` اور تمام ہیشز/commitments **دوبارہ
  حساب** ہوتے ہیں، کبھی دی گئی قیمت سے نہیں لیے جاتے۔

## کنسنسس قواعد

شپ شدہ (`/docs/en/technical/blockchain.md` میں بیان): دستخط
شدہ بلاک پیدا کار (`SCONE-BLOCK-V1` پے لوڈ، چیک پوائنٹ سے منتخب
کمیٹی)، مجموعی چیک پوائنٹس کی فائنلٹی (`SCONE-CKPT-V1`،
فائنلٹی فرش پر فورک چوائس)، نام کی لمبائی کے مطابق رجسٹریشن
PoW۔ باقی موضوعات: باریک اینٹی سپیم فیسز، مالک کلید کی روٹیشن۔

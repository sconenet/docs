# Scone دستاویزات

> `docs/` کے تحت تمام دستاویزات **نافذہ ذریعہ (NORMATIVE)** ہیں: انہیں Rust پڑھے
> بغیر ایک آزاد نفاذ ممکن بنانا چاہیے۔ دستاویز اور کوڈ میں کوئی بھی تضاد
> ایک نقص (bug) ہے (سچائی کا ذریعہ متعین کریں، اسے درست کریں)۔

دو حصے:

- **`general/`** — یہ کیسے کام کرتا ہے: پروٹوکول کے تصورات اور قواعد،
  فارمیٹس کی تفصیلات پڑھے بغیر قابلِ فہم؛
- **`technical/`** — ڈویلپرز کے لیے تفصیلی تفصیلات: درست بائنری فارمیٹس،
  ہیش فارمولے، حدود، APIs، اور ہر جزو کا رویہ۔

## `general/` — یہ کیسے کام کرتا ہے

| دستاویز | مواد |
|---|---|
| [architecture.md](general/architecture.md) | اختیار (blockchain) / دستیابی (DHT) / مقامی اسٹوریج کی علیحدگی؛ crates کے کردار؛ DNS ریزولوشن فلو؛ ارتقاء پذیری |
| [naming.md](general/naming.md) | نام کے قواعد: LDH حروفِ تہجی، TLDs، ڈومینز اور سب ڈومینز، canonicalization، `DomainId` کا حصول اور ہیگزاڈیسمل نمائندگی |

## `technical/` — ڈویلپر تفصیلات

| دستاویز | مواد |
|---|---|
| [protocol.md](technical/protocol.md) | کیننیکل وائر بائنری فارمیٹ: LEB128 varints، شناخت کنندگان، DNS ریکارڈز، بلاکس، P2P پیغامات، حدود، ورژننگ |
| [transactions.md](technical/transactions.md) | ٹرانزیکشن فارمیٹس v1: دستخط شدہ شکلیں (فیلڈز، وائر ترتیب — RegisterDomain نام رکھتا ہے، غیر متعین discriminants، مالک کا حصول، دستخط شدہ پے لوڈ `SCONE-TX-SIG-V1`) اور چار بغیر دستخط pow-to-owner شکلیں (مائن کیا گیا ثبوت ہی اجازت نامہ ہے)، تفویض شدہ اختیار کا خاندان (عدّاد سے منسلک: TransferDomain، آپریٹرز، ریکارڈ delegates)، عدّادِ طفرات، تصدیق کے قواعد |
| [blockchain.md](technical/blockchain.md) | بلاکچین پرت: `TxId`، Merkle درخت، `BlockHash`، جنسیس، کیننیکل اسٹیٹ، بلاک کی تصدیق، کنسنسس تجرید |
| [keystore.md](technical/keystore.md) | `.sconekey` کلیدی فائل فارمیٹ (Argon2id + XChaCha20-Poly1305)، `scone-keystore` API، `identity` کمانڈز |
| [storage.md](technical/storage.md) | مقامی redb استقامت: ٹیبل اسکیما، ایٹمی ڈیلٹاز، میموری پالیسی، اعتماد ماڈل، DoS حدود |
| [relay.md](technical/relay.md) | نیٹ ورک ریلے (`scone-network`): libp2p پروٹوکولز، پیر دریافت (seeds، ڈائریکٹری، UPnP)، پہنچ (AutoNAT، سرکٹ ریلے، DCUtR)، mempool، سنک، Kademlia DHT، مقامی ٹوپولوجی، Prometheus میٹرکس، DNS اسنیپ شاٹ، مکمل CLI، `.env` کنفیگریشن |
| [dns.md](technical/dns.md) | UDP + TCP DNS سرور (M6/M7b): مستند DNS اسنیپ شاٹ، 16 راستے سے شارڈڈ ہاٹ پاتھ کیش (مشترکہ اندراجات، بغیر مختص تلاش، پیک شدہ ریکارڈز، نام کمپریشن)، RFC 1035 کوڈیک، TCP RFC 7766 (512 بائٹ UDP / 64 KiB TCP بجٹ، TC بٹ)، DHT منفی کیش، ریکرسو فال بیک، حدود |
| [cli.md](technical/cli.md) | `scone` بائنری: لاگنگ اور تفصیل (stdout/stderr، `-v` سطریں، `RUST_LOG`)، `.env` کے ذریعے ماحول کنفیگریشن (ترجیح flag > `.env` > ڈیفالٹ)، ریکارڈ delegates (`domain delegates set/clear/list`)، delegate سے دستخط شدہ اپڈیٹس |
| [security-model.md](technical/security-model.md) | سیکیورٹی ماڈل: کمیٹی کا انتخاب، seed اینٹی گرائنڈنگ (تجزیہ: VRF ضروری نہیں)، غلط بیانی اور سلیشنگ |
| [audit-methodologie.md](technical/audit-methodologie.md) | سیکیورٹی آڈٹ کا منصوبۂ کار: تضادات، سچائی کا ذریعہ، درستگی کے مراحل |
| [rfc-proof-of-service.md](technical/rfc-proof-of-service.md) | Proof-of-service RFC: آپریٹر ہارٹ بیٹس، آپریشن کا منتفق ہونا، تازہ سیٹوں میں کمیٹی کا انتخاب |

## کراس ریفرنس روایات

اندرونی دستاویزات اور Rust کوڈ کے تبصرے ان صفحات کا حوالہ ریپوزٹوری روٹ سے
مطلق پاتھ کے ذریعے دیتے ہیں
(مثلاً `/docs/en/technical/protocol.md`)۔

# توثيق Scone

> جميع الوثائق تحت `docs/` هي **معيارية (NORMATIVE)**: يجب أن تتيح
> تنفيذًا مستقلًا دون قراءة كود Rust. أي تباعد بين الوثائق والكود
> يُعد خطأً برمجيًا (حدّد مصدر الحقيقة وصحّحه).

جزءان:

- **`general/`** — كيف يعمل النظام: مفاهيم البروتوكول وقواعده،
  يمكن الوصول إليها دون قراءة تفاصيل الصيغ؛
- **`technical/`** — مواصفات تفصيلية للمطورين: الصيغ الثنائية
  الدقيقة، صيغ الت hashing، الحدود، واجهات API، وسلوك كل مكوّن.

## `general/` — كيف يعمل النظام

| الوثيقة | المحتوى |
|---|---|
| [architecture.md](general/architecture.md) | فصل السلطة (blockchain) / التوفر (DHT) / التخزين المحلي؛ أدوار الـ crates؛ تدفق حلّ DNS؛ قابلية التطور |
| [naming.md](general/naming.md) | قواعد التسمية: أبجدية LDH، نطاقات TLD، النطاقات والنطاقات الفرعية، التوحيد القانوني (canonization)، اشتقاق `DomainId` وتمثيله السداسي عشري |

## `technical/` — مواصفات المطورين

| الوثيقة | المحتوى |
|---|---|
| [protocol.md](technical/protocol.md) | الصيغة الثنائية القانونية للـ wire: varints بترميز LEB128، المعرّفات، سجلات DNS، الكتل، رسائل P2P، الحدود، الإصدارات |
| [transactions.md](technical/transactions.md) | صيغ المعاملات v1: الصيغ الموقّعة (الحقول، ترتيب الـ wire — معاملة RegisterDomain تحمل الاسم، discriminants اعتباطية، اشتقاق المالك، الحمولة الموقّعة `SCONE-TX-SIG-V1`) وصيغ pow-to-owner الأربع بلا توقيع (الإثبات المُستخرَج بالتعدين هو التخويل نفسه)، قواعد التحقق |
| [blockchain.md](technical/blockchain.md) | طبقة البلوكشين: `TxId`، شجرة Merkle، `BlockHash`، كتلة التكوين (genesis)، الحالة القانونية، التحقق من الكتل، تجريد الخوارزمية التوافقية (consensus) |
| [keystore.md](technical/keystore.md) | صيغة ملف المفاتيح `.sconekey` (Argon2id + XChaCha20-Poly1305)، واجهة `scone-keystore`، أوامر `identity` |
| [storage.md](technical/storage.md) | التخزين المحلي الدائم بـ redb: مخطط الجداول، الدلتا الذرية، سياسة الذاكرة، نموذج الثقة، حدود DoS |
| [relay.md](technical/relay.md) | مُرحّل الشبكة (`scone-network`): بروتوكولات libp2p، اكتشاف الأقران (seeds، دليل، UPnP)، قابلية الوصول (AutoNAT، circuit-relay، DCUtR)، الـ mempool، المزامنة، DHT بـ Kademlia، الطوبولوجيا المحلية، مقاييس Prometheus، لقطة DNS، واجهة CLI الكاملة، إعدادات `.env` |
| [dns.md](technical/dns.md) | خادم DNS عبر UDP + TCP (M6/M7b): لقطة DNS مرجعية، ذاكرة تخزين مؤقت ساخنة مقسّمة إلى 16 جزءًا (shards) (مدخلات مشتركة، بحث دون تخصيص ذاكرة، سجلات مضغوطة، ضغط الأسماء)، مرمّز RFC 1035، TCP وفق RFC 7766 (ميزانية 512 بايت لـ UDP / 64 KiB لـ TCP، بت TC)، ذاكرة تخزين سلبية لـ DHT، التراجع التكراري (recursive fallback)، الحدود |
| [cli.md](technical/cli.md) | الملف التنفيذي `scone`: التسجيل (logging) ومستوى التفصيل (stdout/stderr، مستويات `-v`، `RUST_LOG`)، الإعدادات عبر البيئة بواسطة `.env` (الأسبقية: flag > `.env` > الافتراضي) |
| [security-model.md](technical/security-model.md) | نموذج الأمان: اختيار اللجنة، منع الطحن المسبق للبذرة (seed anti-grinding) (تحليل: VRF غير ضروري)، التقابل المزدوج (equivocation) والخصم (slashing) |
| [audit-methodologie.md](technical/audit-methodologie.md) | منهجية التدقيق الأمني: التباعدات، مصدر الحقيقة، جولات التصحيح |
| [rfc-proof-of-service.md](technical/rfc-proof-of-service.md) | RFC إثبات الخدمة (proof-of-service): نبضات المشغّلين (heartbeats)، تفويض العمليات، انتخاب اللجنة من بين المقاعد الطازجة |

## اصطلاحات الإحالة المتبادلة

تشير الوثائق الداخلية وتعليقات كود Rust إلى هذه الصفحات
بالمسار المطلق من جذر المستودع
(مثلاً `/docs/en/technical/protocol.md`).

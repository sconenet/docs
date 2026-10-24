# CLI (`scone`) — التسجيل ومستوى التفصيل

> سلوك معياري لتسجيل الأحداث (logging) في الملف التنفيذي
> `scone`. أي تباعد وثائق/كود = خطأ برمجي (صحّح مصدر الحقيقة).

## المبادئ

- **stdout محجوز لمُخرجات الأوامر** (`scone status`،
  `identity list`، `status`, …). يستطيع سكربت إعادة توجيه
  stdout دون أن يصادف سجلات فيه أبدًا — تظل الأنابيب
  موثوقة. استثناء مقبول: يصدر `scone relay` أسطر البيانات
  `p2p: <multiaddr>` و`dns: <addr>` و`metrics: <addr>` في
  stdout فور ربط الـ listeners
  (بيانات صالحة للاستخدام في `--bootstrap` / الإعداد، لا
  سجلات؛ تصدر مرة واحدة بالضبط لكل منها، مع flush فوري).
- **جميع السجلات تذهب إلى stderr** (صيغة `tracing` المدمجة)،
  بما فيها سجلات المُرحّل — المُرحّل كان يكتب إلى stderr قبل
  الترحيل أصلًا.
- رسائل الخطأ الموجهة للمستخدم من الـ CLI
  (`scone: <error>`) تبقى في stderr: هذه تجربة استخدام، لا
  سجلات.

## المستويات و`--verbose` / `-v`

| الاستدعاء | المستوى الفعلي |
|---|---|
| `scone <cmd>` (دون `-v`) | `warn` |
| `scone relay` (دون `-v`) | `info` |
| `scone -v <cmd>` / `scone relay -v` | `info` |
| `scone -vv …` | `debug` |
| `scone -vvv …` (وأكثر) | `trace` |

الـ `-v`/`--verbose` علم **عام وقابل للتكرار**: يُقبل قبل
ال أمر الفرعي أو بعده (`scone -vv identity list` و
`scone identity list -vv` متكافئان).

**لماذا يبقى `scone relay` في INFO افتراضيًا**: المُرحّل
daemon — يجب أن يسجّل نشاطه (استماع RPC/p2p، الكتل المقبولة
والمنتجَة، الأقران المتصلون، الرفض) ليكون قابلًا للتشغيل،
دون اشتراط علم. أما الأوامر الأحادية فتبقى صامتة (WARN)
كي لا تلوّث طرفية أو سكربتًا.

## `RUST_LOG` (تجاوز)

إذا عُرِّفت متغير البيئة `RUST_LOG`، فهي **تستبدل كليًا**
المستوى المشتق من `-v` (بصيغة `env-filter`):

```bash
RUST_LOG=debug scone identity list        # debug دون -v
RUST_LOG=error scone -vv relay            # error يتغلب على -vv
RUST_LOG=scone_network=trace,info scone relay   # حسب الـ target
```

## الإعداد عبر البيئة (`.env`)

قبل تحليل الوسائط، يحمّل `scone` ملف `.env` من الدليل
الحالي (`dotenvy::dotenv()` في `main.rs`). الملف
الغائب = لا عملية بصمت، وليس خطأً أبدًا. تصبح القيم
المحمّلة خط الأساس لخيارات المُرحّل التي تصرّح بمتغير بيئة
(`env = "…"` في `cli.rs`).

الأسبقية (خيار واحد، ثلاثة مصادر):

```text
علم CLI صريح  >  متغير .env  >  القيمة الافتراضية المدمجة
```

- علم مُمرَّر في سطر الأوامر يفوز دائمًا؛
- وإلا فالمتغير (سواء حُمِّل من `.env` أو عُرِّف في بيئة
  العملية الحقيقية — نفس المتغير، نفس الأثر)؛
- وإلا فالقيمة الافتراضية المدمجة في clap.

المتغيرات القابلة للتكرار (`SCONE_BOOTSTRAP`،
`SCONE_DNS_UPSTREAM`) تتراكم: سطر واحد لكل قيمة في `.env`
(dotenvy يضيف).

الجدول الكامل لمتغيرات `SCONE_*` (الشبكة، الاستماع P2P،
الـ bootstrap، التكوين، UPnP، سطح DNS، المقاييس، ملف
الطوبولوجيا، الـ anchor) ودلالة كل منها يعيش في
[`relay.md`](relay.md#environment-configuration-env)؛ والقالب
المُعلَّق هو `.env.example` في جذر المستودع (انسخه إلى
`.env`، ولا تُدخِله أبدًا في commit — الـ `.env` في
gitignore).

## دلالات المستويات (اصطلاح الكود)

- `error!`: الإخفاقات (اليوم بلا مواضع استدعاء — إخفاقات
  الشبكة تُبتلع كـ `warn!` ولا تظهر إلا الإخفاقات المحلية
  القاتلة كخطأ مُصنَّف من `run()`)؛
- `warn!`: مدخلات شبكة عدائية أو خبيثة مرفوضة، معاملات
  قديمة مُخلاة من الـ mempool، معاملات متعارضة مهملة أثناء
  الإنتاج، إخفاقات صادرة/واردة، عدم تطابق مفاتيح DHT؛
- `info!`: كتلة مقبولة/منتجَة، استماع RPC/p2p، قرين متصل،
  إقلاع المُرحّل (نقاط فحص الحالة)؛
- `debug!`: تفاصيل داخلية (معاملة مقبولة، قرين منفصل، ذهاب
  وإياب RPC، تعداد مخزن المفاتيح، أحداث kad تحت
  `SCONE_KAD_DEBUG`)؛
- `trace!`: محجوز للحمولات (اليوم بلا مواضع استدعاء).

الـ crates المكتبية (`scone-network`, …) تصدر ماكروهات
`tracing` لكنها **لا تهيّئ أي subscriber**: التهيئة تعيش في
الملف التنفيذي `scone` وحده (قبل توزيع الأمر الفرعي —
المُرحّل يحتاج سجلاته منذ الإقلاع).


## الاستخدام البسيط (cli-simplify-v1)

- **دون `--peer`**: تتحدث الأوامر افتراضيًا مع المُرحّل
  المحلي؛ ومتغير البيئة `SCONE_PEER` أو `--peer` يلغيه.
- **دون `--identity`** عندما يحتوي مخزن المفاتيح على هوية
  واحدة (تُستخدم بصمت؛ إن وُجدت عدة هويات ← الخطأ
  يعدّدها).
- محذوفة (الـ stubs تقترح البديل): `show`، `submit`،
  `lookup`، `owned`.
- جديدة: `scone domain renew <name>`، `scone domains`،
  `scone domain get|set` (سجل مفرد، دون ملف).

```bash
scone identity generate me
scone domain register example.scone
scone domain set example.scone A 192.0.2.1
scone domain renew example.scone
```

## دورة حياة الـ TLD (cli-unified-lifecycle)

أمر `scone tld register <tld>` هو نقطة دخول الـ TLD في أمر
واحد، متناظر مع `scone domain register` — تُقرأ الحالة
on-chain أولًا، ثم يُنفَّذ الفعل الصحيح:

- **TLD حر** ← يُسجَّل، ويولد **مفتوحًا افتراضيًا**
  (دون علم: فضاء أسماء ذاتي الخدمة، 1/3 من الـ PoW).
  يختار `--private` فضاء أسماء خاصًا بثمن الـ PoW الكامل، مع
  تأكيد صريح (قرار اقتصادي). يتخطى `--yes` التأكيدات
  (السكربتات).
- **لك، ينتهي خلال أقل من 90 يومًا** ← يُقترح التجديد
  (PoW مرتبط بانتهاء الصلاحية الحالي — منع إعادة الاستخدام)
  ويُنفَّذ بعد التأكيد.
- **لك، بعيد عن الانتهاء** ← لا شيء لفعله (`tld renew
  --force` للتجديد رغم ذلك).
- **لغيرك** ← خطأ مُصنَّف نظيف مع المالك.

```bash
scone tld register mytld            # ولادة مفتوحة (الافتراضي)
scone tld register mytld --private  # فضاء أسماء خاص (تأكيد)
scone tld renew mytld               # تجديد صريح
```

كذلك، `scone domain register <name>` على اسم يخصك ينتهي خلال
أقل من 90 يومًا يقترح التجديد بدلًا من الفشل.

## الأوامر المتقدمة

القيم الافتراضية أعلاه تغطي الاستخدام اليومي. للسكربتات
والتشغيل البيني مع الويب/API تبقى السطحية الكاملة
متاحة:

لكل إجراء مسار أساسي واحد فقط؛ تمت إزالة الأسماء المستعارة الجذرية السابقة (`show` و`lookup` و`owned` و`renew`) (remove-cli-aliases)، وكذلك ذكر `domain info` الذي لم يطابق أي أمر حقيقي قط.

السطحية السداسية غير المتصلة (الأوامر الفرعية `tx
build|sign|verify`، `submit tx`) أُزيلت (remove-tx-surface-v2):
العمليات M8b التي كانت تخدمها صارت أوامر من الدرجة الأولى —
`scone tld open <tld>` (فتح أحادي الجانب ذاتي الخدمة، يُعدّ
لك PoW الفتح)، `scone tld transfer <tld> <owner>`،
`scone tld revoke <tld>`، `scone domain assign <name> <owner>`
(مالك الـ TLD، TLD خاص) و`scone domain transfer
<name> <owner>`. لكل منها: قراءة الحالة، التوقيع، الإرسال،
الـ txid. انتقل حلّ سجلات DNS إلى `scone domain resolve
<name>` (السابق `record get`)؛ وقناة النشر السداسية
(`record put`) اختفت — التحرير يمر عبر `domain set` /
`domain update`.
- `scone domain mine <name>` — يستخرج بالتعدين **فقط** إثبات
  التسجيل (بلا هوية، بلا keystore، بلا إرسال): يطبع JSON
  `scone-domain-pow-v1` `{type, name, network_id, epoch, proof}`
  على stdout. تعدين مفوَّض: آلة أخرى تعدّن لأجلك، مفاتيحك لا
  تغادر جهازك أبدًا. مرِّره عبر أنبوب أو أعِد توجيهه:
  `scone domain mine n.tld | scone domain register n.tld
  --proof -` (أو `--proof FILE`). الإثبات مرتبط بالاسم
  والشبكة؛ والطزاجة يفرضها العقدة عند التطبيق (نافذة صلاحية
  الحقبة). مع `--mine-for <pubkey>` يستخرج بدلًا من ذلك إثبات
  POW-TO-OWNER لذلك المستفيد ويطبع JSON
  `scone-domain-pow-to-v1` `{type, name, network_id, epoch,
  pow_to, proof}` — قابل للاستهلاك **فقط** عبر `register --to` /
  `renew --to` لذلك المفتاح تحديدًا (الـ challenge يتضمّنه؛
  فحص غير حساس لحالة الأحرف عند الإرسال).
- إرسال pow-to-owner (لا حاجة لأي keystore — `--to` حصري مع
  `--identity`): `scone domain register <name> --to <pubkey>` /
  `scone domain renew <name> --to <pubkey>`
  و`scone tld register <tld> --to <pubkey>` / `scone tld renew
  <tld> --to <pubkey>` يرسلون إثباتًا يربط الـ challenge الخاص
  به مفتاح **المستفيد**؛ ويُعاد حساب الـ `owner` على السلسلة
  كـ `hash(pubkey_dest)` (التجديد لا يغيّر الملكية أبدًا). دون
  `--proof` يستخرج الأمر الإثبات بنفسه؛ ومع `--proof -`/FILE
  يستهلك حمولة `scone-domain-pow-to-v1` ويرفض إثباتًا مستخرجًا
  لمستفيد **آخر**. نمط الأنابيب: `scone domain mine n.tld
  --mine-for <pubkey> | scone domain register n.tld --to
  <pubkey> --proof -`.
- `scone domain export <name> [--out FILE]` — يكتب مجموعة
  السجلات المتحقق منها ضد السلسلة بالصيغة الدقيقة لـ
  `update --file` (ذهاب-إياب مضمون: export ← تحرير ←
  update يعيد إرسال نفس المجموعة). دون `--out`: stdout.
  يستخدم `domain resolve` نفس الصيغة المقروءة، لذا فإن
  `resolve > zone.txt` قابل للاستيراد مباشرة. الأنواع بلا
  صياغة نصية (GEOROUTE، غير المعروفة) تُصدَّر كتحذيرات
  مُعلَّقة — لا تُهمَل بصمت أبدًا.

يطبع `domain get` أيضًا الـ `domain_id` و`valid_until`.

## قياس أداء PoW (`scone bench-pow`)

يقيس معدل hashing Argon2id للآلة (نفس الـ hash الذي يستخدمه
تحقق التوافق) ويترجم صعوبات الشبكة إلى أزمنة انتظار ملموسة:

```
scone bench-pow --network mainnet [--seconds 5] [--cores N]
```

الناتج: H/s الكلي ولكل worker، ثم الزمن المتوسط المتوقع لكل
عملية — التنازل/الإسناد (معدل مسطّح)، فتح TLD خاص، وأرضيات
تسجيل TLD/نطاق (منحنى الطول). الشبكة المختارة تحدد معاملات
Argon2id (mainnet: 64 MiB، t=2 — التكلفة الحقيقية؛
testnet/devnet: الملف الشخصي الخفيف، إثباتات بالميلي ثانية).
كل worker يخصّص مسار Argon2id خاصًا به بحجم 64 MiB — اضبط
`--cores` حسب الذاكرة المتاحة.

## البُنى: dev مقابل mainnet مفعّلة (mainnet-compile-gate)

البنى الافتراضية **لا تحمل mainnet**: `scone relay --network
mainnet` (وخدمات المسجّل بـ `SCONE_NETWORK=mainnet`)
تفشل فورًا مع تلميح إعادة البناء. الملف التنفيذي الرسمي
الموزّع يُبنى بـ:

```bash
cargo build --release --features mainnet
```

يعرض `scone --version` علامة `+mainnet` في تلك البنى، ويقول
سجل إقلاع المُرحّل `build: mainnet-enabled`.

يتتبع `--network` الافتراضي البنية أيضًا: `mainnet` في البنى
المفعّلة بالـ mainnet (الملف الرسمي يضع المستخدمين في شبكة
الإنتاج)، و`testnet` خلاف ذلك. علم `--network` صريح أو
متغير البيئة `SCONE_NETWORK` يفوزان دائمًا (ترتيب clap:
flag > env > الافتراضي). إنه حاجز تشغيلي (إقلاعات عرضية،
ملفات dev/CI) — ليس حدًا تشفيريًا؛ قواعد التوافق تبقى
الدفاع ضد العقد العدائية.

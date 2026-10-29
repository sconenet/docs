# CLI (`scone`) — لاگنگ اور تفصیل

> `scone` بائنری کی لاگنگ کا نافذہ رویہ۔ کوئی بھی دستاویز/کوڈ
> تضاد = نقص (سچائی کے ذریعے کا درست کریں)۔

## اصول

- **stdout صرف کمانڈ آؤٹ پٹ کے لیے مخصوص ہے** (`scone status`،
  `identity list`، `status`، …)۔ کوئی اسکرپٹ stdout ری ڈائریکٹ
  کر سکتی ہے اور اس میں کبھی لاگز نہیں ملیں گے — پائپنگ قابلِ
  بھروسہ رہتی ہے۔ منظور شدہ
  استثنا: `scone relay` ڈیٹا لائنیں
  `p2p: <multiaddr>`، `dns: <addr>`، اور `metrics: <addr>`
  stdout پر اسکے طور پر جاری کرتا ہے جیسے ہی سننے والے باندھے جائیں
  (`--bootstrap` / کنفیگریشن کے لیے قابلِ استعمال ڈیٹا، لاگز
  نہیں؛ ہر ایک بالکل ایک بار، فوراً flush)۔
- **تمام لاگز stderr پر جاتے ہیں** (کمپیکٹ `tracing` فارمیٹ)،
  بشمول ریلے کے — ریلے منتقلی سے پہلے ہی stderr پر لاگ کر رہا
  تھا۔
- CLI کے صارف کے سامنے والے خرابی کے پیغامات (`scone: <error>`)
  stderr پر رہتے ہیں: یہ UX ہے، لاگز نہیں۔

## سطریں اور `--verbose` / `-v`

| طلب | مؤثر سطح |
|---|---|
| `scone <cmd>` (`-v` کے بغیر) | `warn` |
| `scone relay` (`-v` کے بغیر) | `info` |
| `scone -v <cmd>` / `scone relay -v` | `info` |
| `scone -vv …` | `debug` |
| `scone -vvv …` (اور اس سے زیادہ) | `trace` |

`-v`/`--verbose` ایک **عالمی اور دہرایا جانے والا** فلیگ ہے: یہ
ذیلی کمانڈ سے پہلے یا بعد میں قبول ہے (`scone -vv identity list`
اور `scone identity list -vv` برابر ہیں)۔

**`scone relay` بطور ڈیفالٹ INFO پر کیوں رہتا ہے**: ریلے ایک
ڈیمن ہے — اسے اپنی سرگرمی کا لاگ کرنا چاہیے (RPC/p2p سننا، قبول شدہ اور
پیدا شدہ بلاکس، جڑے ہوئے پیرز، انکاری) تاکہ چلایا جا سکے،
بغیر کوئی فلیگ مانگے۔ ایک بار کی کمانڈز، دوسری طرف، خاموش
(WARN) رہتی ہیں تاکہ ٹرمینل یا اسکرپٹ کو گندا نہ کریں۔

## `RUST_LOG` (اوور رائیڈ)

اگر `RUST_LOG` ماحول متغیر سیٹ ہو، تو یہ `-v` سے اخذ شدہ سطح
کو **مکمل طور پر بدل دیتا ہے** (`env-filter` نحو):

```bash
RUST_LOG=debug scone identity list        # debug without -v
RUST_LOG=error scone -vv relay            # error wins over -vv
RUST_LOG=scone_network=trace,info scone relay   # per-target
```

## ماحول کنفیگریشن (`.env`)

دلائل پارس کرنے سے پہلے، `scone` موجودہ ڈائریکٹری سے ایک `.env`
فائل لوڈ کرتا ہے (`main.rs` میں `dotenvy::dotenv()`)۔ غائب
فائل = خاموش no-op، کبھی خرابی نہیں۔ لوڈ شدہ قیمتیں ان ریلے
آپشنز کی بنیاد بنتی ہیں جو ماحول متغیر کا اعلان کرتے ہیں
(`cli.rs` میں `env = "…"`)۔

ترجیح (ایک آپشن، تین ذرائع):

```text
explicit CLI flag  >  .env variable  >  built-in default
```

- کمانڈ لائن پر دیا گیا فلیگ ہمیشہ جیتتا ہے؛
- ورنہ متغیر (خواہ `.env` سے لوڈ ہو یا اصل
  پروسیس ماحول میں سیٹ — وہی متغیر، وہی اثر)؛
- ورنہ clap کا بلٹ اِن ڈیفالٹ۔

دہرائے جانے والے متغیرات (`SCONE_BOOTSTRAP`، `SCONE_DNS_UPSTREAM`)
جمع ہوتے ہیں: `.env` میں فی قیمت ایک لائن (dotenvy جوڑتا ہے)۔

`SCONE_*` متغیرات کا مکمل جدول (نیٹ ورک، P2P سننا،
بوٹ اسٹریپ، جنسیس، UPnP، DNS سطح، میٹرکس، ٹوپولوجی
پروفائل، اینکر) اور ہر ایک کے معنی
[`relay.md`](relay.md#environment-configuration-env) میں ہیں؛
تبصرہ شدہ ٹیمپلیٹ ریپوزٹوری روٹ پر `.env.example` ہے (`.env`
میں کاپی کریں، کبھی commit نہ کریں — `.env` gitignored ہے)۔

## سطح کے معنی (کوڈ کی روایت)

- `error!`: ناکامیاں (آج کوئی کال سائٹس نہیں — نیٹ ورک ناکامیاں
  `warn!` کے طور پر نگل لی جاتی ہیں اور صرف مہلک مقامی ناکامیاں
  `run()` سے قسم شدہ خرابی کے طور پر سامنے آتی ہیں)؛
- `warn!`: دور کی گئی معاندانہ یا موقع اندیش نیٹ ورک اندراجات،
  mempool سے نکالے گئے پرانے txs، پیداوار کے دوران چھوڑے گئے
  متضاد txs، آؤٹ باؤنڈ/ان باؤنڈ ناکامیاں، DHT کلید بے میلانی؛
- `info!`: بلاک قبول/پیدا، RPC/p2p سننا، پیر
  جڑا، ریلے اسٹارٹ اپ (اسٹیٹ چیک پوائنٹس)؛
- `debug!`: اندرونی تفصیلات (tx قبول، پیر الگ، RPC
  راؤنڈ ٹرپ، keystore فہرست، `SCONE_KAD_DEBUG` کے تحت kad
  واقعات)؛
- `trace!`: پے لوڈز کے لیے مخصوص (آج کوئی کال سائٹس نہیں)۔

لائبریری crates (`scone-network`، …) `tracing` میکروز جاری کرتی
ہیں مگر **کوئی سبسکرائبر شروع نہیں کرتیں**: ابتدا صرف
`scone` بائنری میں ہے (ذیلی کمانڈ بھیجنے سے پہلے — ریلے کو
بوٹ سے اپنے لاگز چاہئیں)۔


## سادہ استعمال (cli-simplify-v1)

- **کوئی `--peer` ضروری نہیں**: کمانڈز بطور ڈیفالٹ مقامی ریلے سے
  باتیں کرتی ہیں؛ `SCONE_PEER` ماحول یا `--peer` اسے بدلتا ہے۔
- **کوئی `--identity` ضروری نہیں** جب keystore ایک اکیلی
  شناخت رکھتا ہو (چپکے سے استعمال ہوتی ہے؛ متعدد → خرابی انہیں
  فہرست کرتی ہے)۔
- ہٹا دی گئیں (سٹبس تبدیلی تجویز کرتے ہیں): `show`، `submit`،
  `lookup`، `owned`۔
- نئی: `scone domain renew <name>`، `scone domains`، `scone
  domain get|set` (اکیلا ریکارڈ، کوئی فائل نہیں)۔

```bash
scone identity generate me
scone domain register example.scone
scone domain set example.scone A 192.0.2.1
scone domain renew example.scone
```

## TLD زندگی کا چکر (cli-unified-lifecycle)

`scone tld register <tld>` ایک کمانڈ میں TLD کا داخلے کا دروازہ
ہے، `scone domain register` کے متناظر — چین کی اسٹیٹ پہلے
پڑھی جاتی ہے، پھر صحیح عمل چلتا ہے:

- **آزاد TLD** → رجسٹر، بطور ڈیفالٹ **کھل کر پیدا** (کوئی فلیگ
  نہیں: خود سروس نیم اسپیس، PoW کا 1/3)۔ `--private` پوری PoW
  قیمت پر ایک ملکیتی نیم اسپیس چنتا ہے، واضح
  تصدیق کے ساتھ (معاشی فیصلہ)۔ `--yes` تصدیقیں چھوڑ دیتا ہے
  (اسکرپٹس)۔
- **آپ کا، 90 دن میں میعاد ختم** → تجدید (RENEWAL) تجویز ہے
  (موجودہ میعاد سے بندھا PoW — اینٹی ری یوز) اور تصدیق کے بعد
  چلتا ہے۔
- **آپ کا، میعاد سے دور** → کچھ کرنا نہیں (`tld renew --force`
  بہر حال تجدید کے لیے)۔
- **کسی اور کا** → مالک کے ساتھ صاف قسم شدہ خرابی۔

```bash
scone tld register mytld            # open birth (default)
scone tld register mytld --private  # owned namespace (confirm)
scone tld renew mytld               # explicit renewal
```

اسی طرح، آپ کی ملکیت کے اُس نام پر `scone domain register <name>`
جو 90 دن میں ختم ہو ناکامی کے بجائے تجدید تجویز کرتا ہے۔

## ریکارڈ delegates (KSK/ZSK-lite)

ڈومین کا مالک دوسری کلیدوں کو اپنی طرف سے `domain update` پر
دستخط کا حق دے سکتا ہے — ریکارڈ delegate سیٹ (زیادہ سے زیادہ 3
کلیدیں، ہر `set` پر مکمل replacement):

```bash
# منظوری (مالک کے دستخط؛ CLI namespace کی CAS generation
# حاصل کرتا ہے اور ٹرانزیکشن کو اسی سے باندھتا ہے):
scone domain delegates set example.uip \
  --keys <pk-hex-1>,<pk-hex-2> --identity alice

# معائنہ (شناخت کی ضرورت نہیں):
scone domain delegates list example.uip

# سب کچھ منسوخ:
scone domain delegates clear example.uip --identity alice
```

پھر ایک delegate `domain update` بالکل ویسے دستخط کرتا ہے جیسے
مالک کرتا (`--identity deleg`) — آن چین tx کا `owner` فیلڈ
ڈومین کا مالک ہوتا ہے جبکہ دستخط delegate سے آتا ہے۔ منظوری
صرف مالک کے اختیار میں ہے (ایک delegate سیٹ خود نہیں سنبھال
سکتا)، منتقلی اسے صاف کر دیتی ہے، اور ہر `set`/`clear` namespace
کے عدّادِ طفرات کو بڑھاتا ہے (CAS anti-replay — دیکھیں
`transactions.md`): دوبارہ چلایا گیا `set` منسوخ شدہ کلیدوں کو
دوبارہ زندہ نہیں کر سکتا۔ منسوخ شدہ delegate کا اپڈیٹ مقامی طور
پر مسترد ہوتا ہے ("نہ مالک ہے نہ ریکارڈ delegate")۔

## جدید کمانڈز

اوپر کے ڈیفالٹس روزمرہ استعمال کو ڈھانپتے ہیں۔ اسکرپٹس اور ویب/API
باہری عمل کے لیے مکمل سطح موجود رہتی ہے:

ہر عمل کا بالکل ایک معیاری راستہ ہے؛ پرانے روٹ aliases (`show`، `lookup`، `owned`، `renew`) ہٹا دیے گئے ہیں (remove-cli-aliases)، نیز `domain info` کا ذکر بھی، جو کبھی کسی حقیقی کمانڈ سے مطابق نہ تھا۔

آف لائن ہیکس سطح (`tx build|sign|verify`، `submit tx`
ذیلی کمانڈز) ہٹا دی گئی (remove-tx-surface-v2): جن M8b
عملوں کی وہ خدمت کرتی تھی اب پہلے درجے کی کمانڈز ہیں —
`scone tld open <tld>` (یک رخہ خود سروس کھلنا، کھلنے کی
PoW آپ کے لیے مائن)، `scone tld transfer <tld> <owner>`،
`scone tld revoke <tld>`، `scone domain assign <name> <owner>`
(TLD مالک، ذاتی TLD) اور `scone domain transfer <name>`
`<owner>`۔ ہر ایک: اسٹیٹ پڑھنا، دستخط، جمع کروانا، txid۔ DNS ریکارڈ
ریزولوشن `scone domain resolve <name>` میں منتقل (سابقہ `record
get`)؛ ہیکس اشاعت چینل (`record put`) ختم — ترمیم
`domain set` / `domain update` سے ہوتی ہے۔
- `scone domain mine <name>` — صرف رجسٹریشن ثبوت مائن کرتا ہے
  (کوئی شناخت نہیں، کوئی keystore نہیں، کوئی جمع کرانا نہیں):
  `scone-domain-pow-v1` JSON `{type, name, network_id, epoch,
  proof}` stdout پر چھاپتا ہے۔ وفاق شدہ مائننگ: دوسری مشین آپ
  کے لیے مائن کرتی ہے، آپ کی کلیدیں کبھی آپ کے پاس سے نہیں
  نکلتیں۔ پائپ کریں یا ری ڈائریکٹ کریں: `scone domain mine
  n.tld | scone domain register n.tld --proof -` (یا
  `--proof FILE`)۔ ثبوت نام اور نیٹ ورک سے باندھا ہوتا ہے؛
  تازگی نوڈ اپلائی وقت پر نافذ کرتا ہے (ایپوک ولیدیت کھڑکی)۔
  `--mine-for <pubkey>` کے ساتھ وہ اس کے بجائے اُس مستفید کے
  لیے POW-TO-OWNER ثبوت مائن کرتا ہے اور
  `scone-domain-pow-to-v1` JSON `{type, name, network_id,
  epoch, pow_to, proof}` چھاپتا ہے — صرف `register --to` /
  `renew --to` کے ذریعے بالکل اسی کلید کے لیے قابلِ استعمال
  (challenge اُسے اپنے اندر رکھتا ہے؛ جمع کرانے پر کیس
  انسنسیٹو جانچ)۔
- Pow-to-owner جمع کرانا (کسی keystore کی ضرورت نہیں — `--to`
  `--identity` کے ساتھ باہمی استثنا رکھتا ہے): `scone domain
  register <name> --to <pubkey>` / `scone domain renew <name>
  --to <pubkey>` اور `scone tld register <tld> --to <pubkey>` /
  `scone tld renew <tld> --to <pubkey>` ایسا ثبوت جمع کراتے
  ہیں جس کا challenge مستفید کی کلید کو باندھتا ہے؛ آن چین
  `owner` بطور `hash(pubkey_dest)` دوبارہ حساب ہوتا ہے (تجدید
  کبھی ہاتھ نہیں بدلتی)۔ `--proof` کے بغیر کمانڈ خود ثبوت
  مائن کرتی ہے؛ `--proof -`/FILE کے ساتھ وہ
  `scone-domain-pow-to-v1` پے لوڈ استعمال کرتی ہے اور کسی
  دوسرے مستفید کے لیے مائن شدہ ثبوت مسترد کرتی ہے۔ پائپ انداز:
  `scone domain mine n.tld --mine-for <pubkey> | scone domain
  register n.tld --to <pubkey> --proof -`۔
- `scone domain export <name> [--out FILE]` — چین سے تصدیق شدہ
  ریکارڈ سیٹ بالکل `update --file` فارمیٹ میں لکھتا ہے
  (گول ٹرپ کی ضمانت: export → ترمیم → update وہی
  سیٹ دوبارہ جمع کراتا ہے)۔ `--out` کے بغیر: stdout۔ `domain
  resolve` وہی پڑھنے قابل فارمیٹ استعمال کرتا ہے، اس لیے
  `resolve > zone.txt` براہِ راست درآمد قابل ہے۔ جن اقسام کا
  کوئی متنی نحو نہیں (GEOROUTE، نامعلوم) تبصرہ شدہ تنبیہ کے
  طور پر ایکسپورٹ ہوتے ہیں — کبھی چپکے سے نہیں گرائے جاتے۔

`domain get` نیز `domain_id` اور `valid_until` چھاپتا ہے۔

## PoW بینچ مارک (`scone bench-pow`)

مشین کی Argon2id ہیش شرح ناپتا ہے (وہی ہیش جو کنسنسس تصدیق
استعمال کرتی ہے) اور نیٹ ورک کی مشکلوں کا ٹھوس انتظار کے
اوقات میں ترجمہ کرتا ہے:

```
scone bench-pow --network mainnet [--seconds 5] [--cores N]
```

آؤٹ پٹ: کل اور فی ورکر H/s، پھر ہر عمل کا متوقع اوسط وقت —
منتقلی/assign (فلیٹ شرح)، ذاتی TLD کھولنا، اور TLD/ڈومین
رجسٹریشن فرورز (لمبائی وکر)۔ منتخب نیٹ ورک Argon2id
پیرامیٹرز کا تعین کرتا ہے (مین نیٹ: 64 MiB، t=2 — اصل لاگت؛
ٹیسٹ نیٹ/ڈیو نیٹ: ہلکا پروفائل، ملی سیکنڈ ثبوتیں)۔ ہر ورکر
اپنی الگ 64 MiB Argon2id لین مختص کرتا ہے — دستیاب RAM کے
مطابق `--cores` سائز کریں۔

## بلڈز: dev بمقابلہ mainnet-کندہ (mainnet-compile-gate)

ڈیفالٹ بلڈز میں **کوئی مین نیٹ نہیں**: `scone relay --network
mainnet` (اور `SCONE_NETWORK=mainnet` کے ساتھ رجسٹرار خدمات)
فوراً ناکام ہو کر دوبارہ بلڈ کی تجویز دیتی ہیں۔ سرکاری تقسیم شدہ
بائنری اس کے ساتھ بنتی ہے:

```bash
cargo build --release --features mainnet
```

`scone --version` ایسی بلڈز پر `+mainnet` دکھاتا ہے، اور
ریلے اسٹارٹ اپ لاگ کہتا ہے `build: mainnet-enabled`۔

ڈیفالٹ `--network` بھی بلڈ کی پیروی کرتا ہے: mainnet-کندہ
بلڈز پر `mainnet` (سرکاری بائنری صارفین کو پیداواری
نیٹ ورک پر اتارتی ہے)، ورنہ `testnet`۔ کوئی واضح
`--network` فلیگ یا `SCONE_NETWORK` ماحول ہمیشہ جیتتا ہے
(clap ترتیب: فلیگ > ماحول > ڈیفالٹ)۔ یہ ایک
عملی محافظ ریل ہے (اتفاقی چلاؤ، dev/CI بائنریز) — کوئی
خفیہ نگاری کی حد نہیں؛ معاندار نوڈز کے خلاف کنسنسس قواعد ہی
دفاع ہیں۔

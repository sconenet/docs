# CLI (`scone`) — लॉगिंग और वर्बोसिटी

> `scone` बाइनरी के लॉगिंग का नॉर्मेटिव व्यवहार। कोई भी doc/code
> विचलन = बग (सत्य के स्रोत को ठीक करें)।

## सिद्धांत

- **stdout कमांड आउटपुट के लिए आरक्षित है** (`scone status`,
  `identity list`, `status`, …)। कोई स्क्रिप्ट stdout को
  रीडायरेक्ट कर सकती है और उसमें कभी लॉग नहीं मिलेंगे — piping
  विश्वसनीय रहती है। स्वीकृत अपवाद: `scone relay` डेटा पंक्तियाँ
  `p2p: <multiaddr>`, `dns: <addr>`, और `metrics: <addr>` stdout पर
  तोरता है जैसे ही listeners बंध जाते हैं
  (`--bootstrap` / विन्यास के लिए उपयोगी डेटा, लॉग नहीं;
  प्रत्येक ठीक एक बार, तुरंत flush)।
- **सारे लॉग stderr पर जाते हैं** (सुसंहत `tracing` प्रारूप), रिले के
  भी — रिले माइग्रेशन से पहले से ही stderr पर लॉग करता था।
- CLI के उपयोगकर्ता-सम्मुख त्रुटि संदेश (`scone: <error>`)
  stderr पर रहते हैं: यह UX है, लॉग नहीं।

## स्तर और `--verbose` / `-v`

| आह्वान | प्रभावी स्तर |
|---|---|
| `scone <cmd>` (`-v` के बिना) | `warn` |
| `scone relay` (`-v` के बिना) | `info` |
| `scone -v <cmd>` / `scone relay -v` | `info` |
| `scone -vv …` | `debug` |
| `scone -vvv …` (और अधिक) | `trace` |

`-v`/`--verbose` एक **वैश्विक और दोहराने योग्य** फ़्लैग है: यह
subcommand से पहले या बाद में स्वीकृत है (`scone -vv identity list`
और `scone identity list -vv` समतुल्य हैं)।

**`scone relay` डिफ़ॉल्ट रूप से INFO पर क्यों रहता है**: रिले एक
डेमन है — उसे संचालन योग्य होने के लिए अपनी गतिविधि (RPC/p2p
सुनना, स्वीकृत और उत्पादित ब्लॉक, जुड़े पीयर, अस्वीकृतियाँ) लॉग करना
चाहिए, बिना फ़्लैग माँगे। वन-शॉट कमांड इसके विपरीत
मौन (WARN) रहते हैं ताकि टर्मिनल या स्क्रिप्ट को प्रदूषित न करें।

## `RUST_LOG` (override)

यदि `RUST_LOG` environment वेरिएबल सेट है, तो वह `-v` से व्युत्पन्न
स्तर को **पूरी तरह बदल देता है** (`env-filter` सिंटैक्स):

```bash
RUST_LOG=debug scone identity list        # debug without -v
RUST_LOG=error scone -vv relay            # error wins over -vv
RUST_LOG=scone_network=trace,info scone relay   # per-target
```

## वातावरण विन्यास (`.env`)

तर्क पार्स करने से पहले, `scone` मौजूदा निर्देशिका से एक `.env` फ़ाइल
लोड करता है (`main.rs` में `dotenvy::dotenv()`)। फ़ाइल
गायब = मौन no-op, कभी त्रुटि नहीं। लोड किए गए मान उन रिले विकल्पों का
आधार बन जाते हैं जो एक environment वेरिएबल घोषित करते हैं
(cli.rs में `env = "…"`)।

प्राथमिकता (एक विकल्प, तीन स्रोत):

```text
explicit CLI flag  >  .env variable  >  built-in default
```

- कमांड लाइन पर दिया गया फ़्लैग सदा जीतता है;
- अन्यथा वेरिएबल (`.env` से लोड किया हो या वास्तविक
  प्रक्रिया वातावरण में सेट हो — वही वेरिएबल, वही प्रभाव);
- अन्यथा clap का अंतर्निहित डिफ़ॉल्ट।

दोहराने योग्य वेरिएबल (`SCONE_BOOTSTRAP`, `SCONE_DNS_UPSTREAM`)
जमा होते हैं: `.env` में प्रति मान एक पंक्ति (dotenvy append करता है)।

`SCONE_*` वेरिएबलों की पूरी तालिका (नेटवर्क, P2P listen,
bootstrap, genesis, mDNS, UPnP, DNS सतह, metrics, topology
profile, anchor) और प्रत्येक की semantics
[`relay.md`](relay.md#environment-configuration-env) में है;
टिप्पणी किया गया टेम्पलेट रिपॉज़िटरी रूट पर `.env.example` है (इसे
`.env` में कॉपी करें, कभी commit न करें — `.env` gitignored है)।

## स्तर semantics (कोड परंपरा)

- `error!`: विफलताएँ (आज कोई call site नहीं — नेटवर्क विफलताएँ `warn!`
  के रूप में निगली जाती हैं और केवल घातक स्थानीय विफलताएँ `run()` से
  टाइप्ड त्रुटि के रूप में सतह पर आती हैं);
- `warn!`: दूर किए गए शत्रुतापूर्ण या दुर्भावनापूर्ण नेटवर्क प्रविष्टियाँ,
  मेमपूल से निकाले गए बासी txs, उत्पादन के दौरान छोड़े गए
  परस्पर विरोधी txs, outbound/inbound विफलताएँ, DHT कुंजी बेमेल;
- `info!`: ब्लॉक स्वीकृत/उत्पादित, RPC/p2p सुनना, पीयर
  जुड़ा, रिले स्टार्टअप (अवस्था checkpoints);
- `debug!`: आंतरिक विवरण (tx स्वीकृत, पीयर विच्छिन्न, RPC
  round trip, keystore सूचीकरण, `SCONE_KAD_DEBUG` के अंतर्गत kad
  घटनाएँ);
- `trace!`: पेलोड के लिए आरक्षित (आज कोई call site नहीं)।

लाइब्रेरी crates (`scone-network`, …) `tracing` मैक्रोज़ तो तोरते हैं
पर **कोई subscriber आरंभ नहीं करते**: आरंभीकरण केवल `scone`
बाइनरी में रहता है (subcommand dispatch से पहले — रिले को बूट से
ही अपने लॉग चाहिए)।


## सरल उपयोग (cli-simplify-v1)

- **कोई `--peer` ज़रूरी नहीं**: कमांड डिफ़ॉल्ट रूप से स्थानीय रिले से
  बात करते हैं; `SCONE_PEER` env या `--peer` बदल देता है।
- **कोई `--identity` ज़रूरी नहीं** जब कीस्टोर में एक ही पहचान हो (मौन
  रूप से उपयोग; कई → त्रुटि उन्हें सूचीबद्ध करती है)।
- हटाए गए (stubs विकल्प सुझाते हैं): `show`, `submit`,
  `lookup`, `owned`।
- नए: `scone domain renew <name>`, `scone domains`, `scone domain
  get|set` (एकल रिकॉर्ड, कोई फ़ाइल नहीं)।

```bash
scone identity generate me
scone domain register example.scone
scone domain set example.scone A 192.0.2.1
scone domain renew example.scone
```

## TLD जीवनचक्र (cli-unified-lifecycle)

`scone tld register <tld>` एक-कमांड TLD प्रवेश बिंदु है,
`scone domain register` के सममित — पहले ऑन-चेन अवस्था
पढ़ी जाती है, फिर सही क्रिया चलती है:

- **खाली TLD** → पंजीकृत, **डिफ़ॉल्ट रूप से खुला जन्म** (कोई फ़्लैग
  नहीं: स्व-सेवा नेमस्पेस, PoW का 1/3)। `--private` पूर
  PoW कीमत पर स्वामित्व वाला नेमस्पेस चुनता है, स्पष्ट
  पुष्टि के साथ (आर्थिक निर्णय)। `--yes` पुष्टियाँ छोड़ता है
  (स्क्रिप्ट)।
- **आपका, 90 दिनों में समाप्त** → नवीनीकरण प्रस्तावित
  (वर्तमान समाप्ति से बंधी PoW — एंटी-पुनःउपयोग) और पुष्टि के
  बाद चलता है।
- **आपका, समाप्ति से दूर** → कुछ करने को नहीं (`tld renew --force`
  से फिर भी नवीनीकरण)।
- **किसी और का** → स्वामी के साथ साफ़ टाइप्ड त्रुटि।

```bash
scone tld register mytld            # open birth (default)
scone tld register mytld --private  # owned namespace (confirm)
scone tld renew mytld               # explicit renewal
```

इसी तरह, अपने किसी नाम पर `scone domain register <name>` जो 90
दिनों में समाप्त हो, विफल होने के बजाय नवीनीकरण प्रस्तावित करता है।

## उन्नत कमांड

ऊपर के डिफ़ॉल्ट रोज़मर्रा उपयोग को कवर करते हैं। स्क्रिप्ट और web/API
इंटरऑप के लिए पूरी सतह उपलब्ध रहती है:

प्रत्येक क्रिया का ठीक एक ही मानक पथ है; पुराने रूट उपनाम (`show`,
`lookup`, `owned`, `renew`) हटा दिए गए हैं (remove-cli-aliases),
साथ ही `domain info` का उल्लेख भी, जो कभी किसी वास्तविक कमांड से
मेल नहीं खाता था।

ऑफ़लाइन हेक्स सतह (`tx build|sign|verify`, `submit tx`
subcommands) हटा दी गई (remove-tx-surface-v2): जिन M8b
संचालनों के लिए वह थी वे अब प्रथम-श्रेणी कमांड हैं —
`scone tld open <tld>` (एक-तरफ़ा स्व-सेवा खोलना, खोलने की
PoW आपके लिए माइन), `scone tld transfer <tld> <owner>`,
`scone tld revoke <tld>`, `scone domain assign <name> <owner>`
(TLD स्वामी, निजी TLD) और `scone domain transfer <name>
<owner>`। प्रत्येक: अवस्था पढ़ना, हस्ताक्षर, जमा, txid। DNS रिकॉर्ड
रिज़ॉल्यूशन `scone domain resolve <name>` में चला गया (पूर्व-`record
get`); हेक्स प्रकाशन चैनल (`record put`) गायब —
संपादन `domain set` / `domain update` से होता है।
- `scone domain export <name> [--out FILE]` — चेन-सत्यापित
  रिकॉर्ड समुच्चय ठीक `update --file` प्रारूप में लिखता है
  (round-trip गारंटीड: export → edit → update वही
  समुच्चय पुनः जमा करता है)। `--out` के बिना: stdout। `domain resolve` वही
  पठनीय प्रारूप उपयोग करता है, इसलिए `resolve > zone.txt` सीधे
  आयात योग्य है। जिन प्रकारों का कोई पाठ्य सिंटैक्स नहीं (GEOROUTE, unknown)
  वे टिप्पणी की गई चेतावनियों के रूप में निर्यात होते हैं — कभी मौन से नहीं छोड़े जाते।

`domain get` साथ ही `domain_id` और `valid_until` छापता है।

## Builds: dev बनाम mainnet-सक्षम (mainnet-compile-gate)

डिफ़ॉल्ट builds में **कोई mainnet नहीं**: `scone relay --network
mainnet` (और `SCONE_NETWORK=mainnet` वाले registrar services)
तुरंत विफल होते हैं rebuild संकेत के साथ। आधिकारिक वितरित
बाइनरी इससे बनती है:

```bash
cargo build --release --features mainnet
```

ऐसी builds पर `scone --version` `+mainnet` दिखाता है, और
रिले स्टार्टअप लॉग `build: mainnet-enabled` कहता है।

डिफ़ॉल्ट `--network` भी build का अनुसरण करता है: mainnet-सक्षम
builds पर `mainnet` (आधिकारिक बाइनरी उपयोगकर्ताओं को
उत्पादन नेटवर्क पर उतारती है), अन्यथा `testnet`। स्पष्ट
`--network` फ़्लैग या `SCONE_NETWORK` env सदा जीतता है
(clap क्रम: flag > env > default)। यह एक
संचालन रेलिंग है (दुर्घटनात्मक चलाने, dev/CI बाइनरी) — कोई
क्रिप्टोग्राफ़िक सीमा नहीं; शत्रुतापूर्ण नोड्स के विरुद्ध सहमति नियम ही
बचाव बने रहते हैं।

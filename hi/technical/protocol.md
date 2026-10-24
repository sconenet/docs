# प्रोटोकॉल

सकोन प्रोटोकॉल के लिए संदर्भ दस्तावेज़: बाइनरी प्रारूप, कैननिकल
एनकोडिंग, लेनदेन, DNS रिकॉर्ड, ब्लॉक, P2P संदेश, सीमाएँ और
संस्करण संगति। संदर्भ कार्यान्वयन
[`scone-protocol`](../../../crates/scone-protocol) crate है।

**(अस्थायी)** चिह्नित मद अंतिम नहीं हैं।

## प्रोटोकॉल संस्करण

- स्थिरांक: `PROTOCOL_VERSION: u32 = 1` (`scone-protocol`);
- `Hello.version` (हैंडशेक) और `BlockHeader.version` द्वारा वहन किया जाता है;
- संगति नियम:
  - प्राप्त संस्करण स्थानीय संस्करण से **अधिक** → अस्वीकृत
    (`ProtocolError::UnsupportedVersion`); नोड जिस संस्करण को नहीं जानता उसका
    प्रारूप अनुमान नहीं लगाता;
  - संस्करण `0` → अमान्य;
  - केवल एक ही प्रारूप-समुच्चय है: लेनदेन संस्करण बाइट्स
    (`0x01`) और ब्लॉक संस्करण को स्थानीय मानों से **ठीक-ठीक** मेल
    खाना चाहिए;
- इस चरण में कोई major/minor विभाजन नहीं: एक ही पूर्णांक, किसी भी
  वायर प्रारूप टूट पर बढ़ाया जाता है।

## बाइनरी प्रारूप सिद्धांत

- बाइनरी, प्रोटोकॉल डेटा के लिए कभी JSON नहीं;
- नियतिवादी और कैननिकल: तार्किक रूप से समान दो वस्तुएँ **ठीक वही बाइट्स**
  उत्पन्न करती हैं;
- क्रमिक पार्सिंग, जहाँ संभव हो बिना आवंटन (बाइट borrows);
- सारा नेटवर्क इनपुट अविश्वसनीय है: लंबाई और काउंटर किसी भी
  आवंटन से **पहले** जाँचे जाते हैं; डिकोडिंग कभी panic नहीं करती।

### एंडियननेस

- सारे पूर्णांक LEB128 varint हैं (7-बिट समूह, अल्प महत्वपूर्ण
  पहले): देशी एंडियननेस लागू नहीं होती;
- IPv4/IPv6 पते और 32-बाइट पहचानकर्ता: कच्चे बाइट सरणियाँ, कोई
  लंबाई प्रीफ़िक्स नहीं;
- प्रारूप में कोई निश्चित-आकार विस्तृत पूर्णांक नहीं: एक ही पूर्णांक
  एनकोडिंग नियम।

### Varint (न्यूनतम LEB128)

प्रत्येक बाइट 7 बिट डेटा वहन करती है; अंतिम के अतिरिक्त प्रत्येक बाइट पर
हाई बिट 1 है। एक `u64` 1 से 10 बाइट में फिट होता है।

| मान | बाइट्स (हेक्स) |
|---|---|
| 0 | `00` |
| 1 | `01` |
| 127 | `7f` |
| 128 | `80 01` |
| 255 | `ff 01` |
| 16,384 | `80 80 01` |
| 2^32 | `80 80 80 80 10` |
| u64::MAX | `ff ff ff ff ff ff ff ff ff 01` |

एनकोडिंग **न्यूनतम** होनी चाहिए: अति-आकार वाले रूप डिकोडिंग पर
अस्वीकृत होते हैं (`InvalidVarint`)। यह उस आवश्यक शर्त के लिए है कि
एक मान की केवल एक ही वैध बाइट अनुक्रमणिका हो (कैननिकल एनकोडिंग)।
अस्वीकृतियाँ: `80 00` (गैर-न्यूनतम), `ff×9 02` (u64 ओवरफ़्लो), `ff×10`
(बहुत लंबा)।

### स्ट्रिंग और बाइट्स

- `bytes`: varint लंबाई + कच्चे बाइट्स;
- `str`: वही, अनिवार्य UTF-8 डिकोडिंग के साथ;
- प्रत्येक लंबाई सीमित है (देखें [सीमाएँ](#सीमाएँ)) और पढ़ने से पहले
  जाँची जाती है।

### पहचानकर्ता

| प्रकार | वायर प्रतिनिधित्व |
|---|---|
| `DomainId` | 32 कच्चे बाइट्स |
| `TldId` | 32 कच्चे बाइट्स |
| `OwnerId` | 32 कच्चे बाइट्स |
| `PublicKeyRef` | 32 कच्चे बाइट्स |
| `RecordHash` | 32 कच्चे बाइट्स |
| `BlockHash` | 32 कच्चे बाइट्स |
| `MerkleRoot` | 32 कच्चे बाइट्स |
| `DomainName` | `str` (≤ 253 बाइट, `scone-core` द्वारा पुनः सत्यापित) |

व्युत्पत्तियाँ (`BLAKE3-256` + `SCONE-DOMAIN-V1` प्रीफ़िक्स, इत्यादि)
`scone-core` द्वारा ही परिभाषित रहती हैं (देखें `/docs/en/general/naming.md`)।

## लेनदेन

```text
Transaction = disc u8 || version u8 || payload
```

| डिस्क्रिमिनेंट | प्रकार |
|---|---|
| `0x21` | `RegisterDomain` |
| `0x2A` | `RegisterTldPow` |
| `0x52` | `UpdateDomain` |
| `0x57` | `RenewTldPow` |
| `0x84` | `RegisterDomainPow` |
| `0x93` | `RegisterTld` |
| `0xC3` | `RenewDomainPow` |

प्रत्येक डिस्क्रिमिनेंट एक मनमाना अपारदर्शी स्थिरांक है (कोई तार्किक
क्रम नहीं; `0x00` कभी वैध नहीं) — पूरी नॉर्मेटिव तालिका
`/docs/en/technical/transactions.md` में।

`version` बाइट (वर्तमान मान `0x01`) डिस्क्रिमिनेंट का अनुसरण करती है:
कोई अन्य मान स्पष्ट रूप से अस्वीकृत है
(`UnsupportedVersion(value)`)। अधिकांश लेनदेन हस्ताक्षरित हैं
(Ed25519) — चार बिना-हस्ताक्षर pow-to-owner रूपों
(`0x2A`, `0x57`, `0x84`, `0xC3`) को छोड़कर: माइन किया गया प्रूफ ही
प्राधिकरण है, और उसका challenge लाभार्थी कुंजी और नेटवर्क को
बाँधता है। पूरा विनिर्देश, हस्ताक्षरित पेलोड और सत्यापन नियम
**`/docs/en/technical/transactions.md`** (नॉर्मेटिव) में।

2026-09 वायर परिवर्तन के बाद से, **लेनदेन अब `network` फ़ील्ड
नहीं रखते**: `network_id` (`str` ≤ 16) **हस्ताक्षर पेलोड**
में (हस्ताक्षरित रूप) या **PoW challenge** में (pow-to-owner रूप)
इंजेक्ट होता है — एक नेटवर्क के लिए बनाया गया tx कभी
दूसरे पर नहीं जाता (देखें `/docs/en/technical/blockchain.md`)।

### REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

`RegisterDomain` **कैननिकल नाम स्पष्ट पाठ में** वहन करता है, उसके
बाद उसका `DomainId` — संगति `domain_id == DomainId(name)` डिकोडिंग पर
पुनः जाँची जाती है (पुनर्गणित, कभी विश्वास नहीं)। `proof` रजिस्ट्रेशन
PoW वहन करता है (20 बाइट: nonce + epoch + difficulty — epoch
साल्ट `epoch = height/144` लागू किए जाने वाले ब्लॉक का, नेटवर्क स्थिरांक और
ब्लॉक के epoch के विरुद्ध अनुप्रयोग पर सख्ती से सत्यापित; बेमेल पर टाइप्ड
अस्वीकृति `ProofEpochMismatch` — M8b, देखें
`transactions.md`)। `public_key` हस्ताक्षरकर्ता की Ed25519 कुंजी है (ठीक
32 बाइट); `owner` उस कुंजी की BLAKE3 व्युत्पत्ति होनी चाहिए
(पुनर्गणित, कभी विश्वास नहीं); `signature` ठीक 64 बाइट है।

### REGISTER_TLD (0x93)

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

TLD का दावा (डोमेन रजिस्ट्री से पृथक रजिस्ट्री, M7): TLD
एक डोमेन के नाम की तरह **स्पष्ट पाठ में वहन** किया जाता है (M8b — PoW
चुनौती नाम से व्युत्पन्न होती है), और `tld_id` उस नाम की `SCONE-TLD-V1`
व्युत्पत्ति है, जो निर्माण ही से प्रत्येक `DomainId` से असंबद्ध है। अवस्था
नियम (M8b): TLD खाली + ICANN-आरक्षित नहीं
(`IcannTldReserved` — ICANN रूट TLD किसी के भी द्वारा दावा योग्य नहीं) + PoW
लेबल की **लंबाई-आधारित** कठिनाई पर (`pow_length` वक्र —
देखें `blockchain.md`); दावा किया गया बंद रहता है (केवल-assign)।

### UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

डिकोडिंग पर पुनः जाँचा जाने वाला अपरिवर्तनीय नियम: `sequence > 0` और `owner` ==
`public_key` की व्युत्पत्ति। कोई `timestamp` नहीं: क्रमण
`sequence` से होता है। लेनदेन केवल सुसंगत संदर्भ वहन करता है — पूरी
DNS सामग्री DHT में रहती है (नीचे देखें)।

## DNS रिकॉर्ड (DHT पक्ष)

पूरे DNS रिकॉर्ड कभी ब्लॉकचेन में प्रवेश नहीं करते: वे DHT के माध्यम से
विनिमय होते हैं (`GetRecord`/`Record` संदेश) और ऑन-चेन केवल उनके
`RecordHash` द्वारा प्रतिबद्ध होते हैं।

### RecordData

```text
RecordData = type.v || ttl.v || payload(type)
```

प्रकार कोड **वास्तविक DNS कोड** (IANA) हैं। TTL (सेकंड,
varint) प्रकार कोड का अनुसरण करता है और पूरे रिकॉर्ड को कवर करता है: सीमाएँ
`MIN_TTL = 1` .. `MAX_TTL = 2,592,000` (30 दिन; डिफ़ॉल्ट
`DEFAULT_TTL = 86,400`)।

| कोड | प्रकार | पेलोड |
|---|---|---|
| 1 | A | 4 बाइट |
| 2 | NS | `name` |
| 5 | CNAME | `name` |
| 12 | PTR | `name` |
| 15 | MX | `preference.v`, `name` |
| 16 | TXT | `str` ≤ 4096 |
| 28 | AAAA | 16 बाइट |
| 33 | SRV | `priority.v`, `weight.v`, `port.v`, `target(name)` |
| 257 | CAA | `flags u8`, `tag` (1–15 B), `value` (`str` ≤ 4096) |
| 32768 | GEOROUTE | GeoRoute (नीचे देखें) |
| अन्य | Unknown | code.v + `bytes` ≤ 4096 |

उदाहरण: `A 192.0.2.1` (TTL 86,400) → `01 80 a3 05 c0 00 02 01`
(प्रकार, TTL varint, पता)।

कोई ज्ञात कोड वहन करने वाला `Unknown` एनकोडिंग पर अस्वीकृत होता है
(डिकोडिंग कभी ऐसा उत्पन्न नहीं कर सकती)।

#### GEOROUTE (GeoDNS)

निजी सकोन प्रकार (गैर-IANA श्रेणी)। एक भौगोलिक रूटिंग नियम:

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

**कैननिकल नियम**: किसी दिए गए scope के endpoints
**क्रमबद्ध** एनकोड किए जाते हैं (priority आरोही, फिर weight अवरोही, फिर target
आरोही); क्रमबद्धता एनकोडिंग पर लागू होती है और अक्रमबद्ध वायर डिकोडिंग पर
अस्वीकृत होती है (`NonCanonical`), रिकॉर्ड समुच्चयों की तरह। पूर्ण
endpoint प्रतिकृतियाँ अस्वीकृत होती हैं।

**सत्यापन**: `priority > 0`, `weight ≤ 100` (0 = निष्क्रिय
endpoint), प्रति scope ≤ 32 endpoints (`GEO_MAX_ENDPOINTS`)। स्थानीय
तालिका के लिए अज्ञात देश/महाद्वीप कोड
(`continent_of_country`) वायर स्तर पर **स्वीकृत** है: रिज़ॉल्वर की
तालिका प्रकाशक की तालिका से अधिक नई हो सकती है। TTL
जनक `RecordData` का ही है (अपरिवर्तित)।

### DnsRecord

```text
domain_id[32] sequence.v expiration.v count.v(≤ 256) records…
```

**कैननिकल नियम**: रिकॉर्ड समुच्चय एक अक्रमित संग्रह है। इसे
**प्रत्येक रिकॉर्ड की व्यक्तिगत एनकोडिंग की शब्दकोशीय क्रम से क्रमबद्ध**
एनकोड किया जाता है, सख्ती से आरोही (प्रतिकृतियाँ अस्वीकृत)। अक्रमबद्ध
वायर डिकोडिंग पर अस्वीकृत होती है (`NonCanonical`)। मूल `Vec` का
क्रम परिवहित नहीं होता।

### SignedDnsRecord

```text
DnsRecord owner[32] signature(bytes ≤ 128)
```

### RecordHash

```text
RecordHash = BLAKE3-256("SCONE-RECORD-V1" || canonical(DnsRecord))
```

हैश **केवल रिकॉर्ड सामग्री** को कवर करता है; स्वामी का हस्ताक्षर
उन्हीं कैननिकल बाइट्स पर अलग से सत्यापित होता है। *(निर्णय: प्रतिबद्धता
हस्ताक्षर को कवर नहीं करती — इसी दस्तावेज़ के एक पुराने संस्करण में
"पूर्ण SignedDnsRecord" लिखा था; हैश को हस्ताक्षर से जोड़ने से कुछ नहीं
मिलता था और प्रतिबद्धता हस्ताक्षर योजना से बंध जाती थी।)*

## ब्लॉक

### BlockHeader

```text
version.v height.v prev_hash[32] tx_root[32] timestamp.v consensus(bytes ≤ 65,536)
```

- `version`: प्रोटोकॉल संस्करण (ऊपर के नियम);
- `height`: ऊँचाई (genesis = 0);
- `prev_hash`: पिछले ब्लॉक का हैश (genesis ब्लॉक के लिए शून्य);
- `tx_root`: लेनदेन सूची पर प्रतिबद्धता (मर्कल ट्री जो
  `scone-blockchain` द्वारा परिभाषित है, देखें `/docs/en/technical/blockchain.md`);
- `timestamp`: क्रम सूचना (Unix, सेकंड);
- `consensus`: उत्पादक पेलोड (`SCONE-BLOCK-V1 ‖ pk[32] ‖
  sig[64]`, देखें `blockchain.md`) — किसी नेटवर्क का genesis कच्चा
  `network_id` वहन करता है, और आवंटन genesis (मेननेट किकस्टार्टर) एक
  `ALLOC-V1` मैनिफ़ेस्ट (≤ 60 KiB) वहन कर सकता है; इसे
  `MAX_CONSENSUS_LEN` (64 KiB) सीमित करता है।

### Block

```text
BlockHeader tx_count.v(≤ 4096) transactions…
```

क्रमिक पार्सिंग; काउंटर आवंटन से **पहले** सीमित है; कोई भी
कर्तन अस्वीकृत होती है। रिकॉर्ड समुच्चयों के विपरीत, लेनदेन क्रम
**महत्वपूर्ण** (सहमति क्रम) है और ठीक-ठीक संरक्षित रहता है।

`BlockHash` (कैननिकल हेडर का हैश) और `tx_root` यहाँ परिभाषित
एनकोडिंग पर `scone-blockchain` द्वारा गणना करते हैं (सटीक सूत्र
`/docs/en/technical/blockchain.md` में)।

## P2P संदेश

```text
Message = disc u8 || payload
```

परिवहन (फ़्रेमिंग, libp2p…) कार्यक्षेत्र से बाहर है; एक परिवहन फ़्रेम
`MAX_MESSAGE_LEN` (1 MiB) से अधिक नहीं होना चाहिए।

| Disc | संदेश | दिशा | पेलोड | सीमा | अपेक्षित उत्तर |
|---|---|---|---|---|---|
| `0x01` | `Hello` | दोनों | `version.v` | — | `Hello` |
| `0x02` | `Ping` | दोनों | — | — | `Pong` |
| `0x03` | `Pong` | दोनों | — | — | — |
| `0x04` | `GetBlock` | दोनों | `hash[32]` | — | `Block` |
| `0x05` | `GetBlocks` | दोनों | `start.v` `max.v` | `max ≤ 128` | ≤ `max` × `Block` |
| `0x06` | `Block` | दोनों | `Block` | tx ≤ 4096 | — |
| `0x07` | `Transaction` | दोनों | `Transaction` | — | — |
| `0x08` | `GetRecord` | दोनों | `domain_id[32]` | — | `Record` |
| `0x09` | `Record` | दोनों | `SignedDnsRecord` | — | — |
| `0x0a` | `Checkpoint` | दोनों | `Checkpoint` | signers ≤ 64 | — |
| `0x0b` | `GetCheckpoints` | दोनों | — | — | checkpoint window |
| `0x0c` | `GetStatus` | दोनों | — | — | `NodeStatus` |
| `0x0d` | `NodeStatus` | दोनों | नोड अवस्था (network, tip, height, NAT, domains, mempool, peers, finality, fresh operators) | — | — |
| `0x0e` | `LookupName` | दोनों | `name(str)` | — | `NameState` |
| `0x0f` | `NameState` | दोनों | नाम + TLD की ऑन-चेन अवस्था | — | — |
| `0x10` | `GetOwnedNames` | दोनों | `owner[32]` | — | `OwnedNames` |
| `0x11` | `OwnedNames` | दोनों | स्वामी के जीवित डोमेन + TLD | — | — |
| `0x12` | `Error` | उत्तर | अनुप्रयोग-स्तरीय त्रुटि संदेश (`str`) | — | — |

### Checkpoint (M5)

एक समुच्चित चेकपॉइंट वायर पर इस रूप में जाता है: टैग `SCONE-CKPT-V1` ||
बॉडी `CheckpointData` (116 B, LE — हस्ताक्षर प्रीइमेज के अंत के समान:
`epoch(8) height(8) block_hash(32) prev_checkpoint_hash(32)
state_root(32) recovery(4)`) || `count.v` || प्रत्येक हस्ताक्षरकर्ता के लिए
`pk[32] ‖ sig[64]`, **आरोही pk क्रम में, बिना प्रतिकृति**, `count ≤ 64`
(`MAX_CHECKPOINT_SIGNERS`)। सख्त डिकोडिंग: कर्तन, गलत टैग,
गैर-कैननिकल pk, प्रतिकृतियाँ या अक्रमबद्ध = अस्वीकृति (`None`), कभी
panic नहीं (भ्रष्टाचार फ़ज़ परीक्षित)।

`GetCheckpoints` फ़ाइनलिटी विंडो का अनुरोध करता है; उत्तर एक या अधिक
`Checkpoint` संदेशों के माध्यम से `checkpoint_window()` विंडो (अधिकतम
`CHECKPOINT_KEEP` चेकपॉइंट, पुराने से नए) है।

`TxId` अब `scone-blockchain` द्वारा परिभाषित है
(`BLAKE3-256("SCONE-TX-V1" || canonical(Transaction))`, देखें
`/docs/en/technical/blockchain.md`)। मेमपूल और विंडोड
एंटी-रीप्ले (`TxReplay`, `REPLAY_WINDOW_BLOCKS = 256`) रिले में
शिप किए गए हैं (M4); केवल `GetTransaction` अब भी टाला गया है (इसकी
परिभाषा शेष सहमति निर्णयों पर निर्भर है)।

## ब्लॉकचेन / DHT विभाजन

```text
BLOCKCHAIN                      DHT
──────────                      ───
domain_id                       domain_id
owner                           SignedDnsRecord (full, signed)
sequence
record_hash
```

चेन कहती है *किसके* पास है और *वर्तमान अवस्था क्या है* (sequence +
record_hash)। DHT *सामग्री* देता है। एक DHT रिकॉर्ड तभी स्वीकृत होता है जब:

1. उसका हस्ताक्षर चेन पर पंजीकृत स्वामी के लिए वैध हो;
2. उसका `sequence` चेन के `sequence` के अनुरूप (या उससे ऊँचा) हो;
3. उसकी कैननिकल एनकोडिंग पर पुनर्गणित `RecordHash` चेन के
   `record_hash` से मेल खाता हो।

## सीमाएँ

| स्थिरांक | मान | भूमिका |
|---|---|---|
| `MAX_NAME_LEN` | 253 | नाम की लंबाई |
| `MAX_TLD_LEN` | 63 | TLD की लंबाई |
| `MAX_TXT_LEN` | 4096 | TXT स्ट्रिंग |
| `MAX_PROOF_LEN` | 256 | रजिस्ट्रेशन प्रमाण |
| `MAX_SIGNATURE_LEN` | 128 | हस्ताक्षर (Ed25519 = 64) |
| `MAX_RECORDS_PER_SET` | 256 | प्रति समुच्चय रिकॉर्ड |
| `MAX_DHT_RECORD_LEN` | 64 KiB | एनकोडेड हस्ताक्षरित DNS रिकॉर्ड (DHT पेलोड: wire, storage, gossip) — कुल सीमा एनकोडिंग पर और प्रत्येक डिकोडिंग से पहले जाँची जाती है; किसी यथार्थ समुच्चय से 15–30× ऊपर (SPF+DKIM+DMARC ≈ 2–5 KiB), परिवहन सीमा से 16× नीचे। |
| `GEO_MAX_ENDPOINTS` | 32 | प्रति GeoRoute scope endpoints |
| `MAX_UNKNOWN_DATA` | 4096 | अज्ञात प्रकार का डेटा |
| `MAX_TXS_PER_BLOCK` | 4096 | प्रति ब्लॉक लेनदेन |
| `MAX_CONSENSUS_LEN` | 64 KiB | हेडर का सहमति पेलोड (`ALLOC-V1` genesis मैनिफ़ेस्ट के लिए चौड़ा किया गया — मेननेट किकस्टार्टर) |
| `MAX_BLOCKS_PER_REQUEST` | 128 | प्रति `GetBlocks` ब्लॉक |
| `MAX_MESSAGE_LEN` | 1 MiB | परिवहन फ़्रेम (अनुशंसित) |

कोई भी अधिकता → `LimitExceeded`, आवंटन से पहले पता चलती है: दुर्भावनापूर्ण
पीयर डिकोडिंग पर असंगत आवंटन नहीं करवा सकता।

## कैननिकल एनकोडिंग

मूल नियम:

```text
canonical object → canonical binary encoding → hash / signature
```

- केवल न्यूनतम varint;
- रिकॉर्ड समुच्चय सख्ती से आरोही क्रमबद्ध;
- तार्किक रूप से समान वस्तु की ठीक **एक** वैध एनकोडिंग होती है;
- डिकोडिंग गैर-कैननिकल रूपों को अस्वीकार करती है (`NonCanonical`), इसलिए
  एक ही वस्तु के लिए कई वैध एनकोडिंग नहीं होते।

## त्रुटि प्रबंधन

`ProtocolError` (`scone-protocol::error`):

| वेरिएंट | अर्थ |
|---|---|
| `Truncated` | इनपुट किसी मान के बीच में समाप्त |
| `InvalidVarint` | गैर-न्यूनतम, ओवरफ़्लोइंग या बहुत लंबा varint |
| `IntegerOutOfRange` | पूर्णांक अपने लक्ष्य प्रकार में फिट नहीं |
| `UnsupportedVersion` | शून्य या स्थानीय से ऊँचा संस्करण |
| `UnknownDiscriminant` | अज्ञात लेनदेन/संदेश डिस्क्रिमिनेंट |
| `LimitExceeded` | सीमा से परे लंबाई या गिनती |
| `InvalidUtf8` | गैर-UTF-8 स्ट्रिंग |
| `TrailingBytes` | पूर्ण मान के बाद शेष बाइट्स |
| `Validation` | किसी `scone-core` अपरिवर्तनीय नियम का उल्लंघन |
| `NonCanonical` | प्रति-फ़ील्ड वैध लेकिन गैर-कैननिकल एनकोडिंग |

नेटवर्क डेटा पर डिकोडिंग कभी panic नहीं करती।

## हस्ताक्षर

- योजना: **Ed25519** (RFC 8032), `scone-crypto` के माध्यम से
  (`ed25519-dalek`, `verify_strict` सत्यापन — मैलिएबल
  हस्ताक्षर और दुर्बल कुंजियाँ अस्वीकृत);
- **चेन लेनदेन**: प्रत्येक `RegisterDomain` / `RegisterTld` /
  `UpdateDomain` पेलोड `"SCONE-TX-SIG-V1" || canonical_encode(tx_without_signature)` पर
  `public_key` (32 B) + `signature` (64 B) एम्बेड करता है;
  पूरा नॉर्मेटिव विनिर्देश: `/docs/en/technical/transactions.md`;
- **DHT रिकॉर्ड** (`SignedDnsRecord`): हस्ताक्षरित पेलोड
  `DnsRecord` की कैननिकल एनकोडिंग है; सत्यापन `record + owner + signature`
  को ब्लॉकचेन में पंजीकृत पहचान के साथ क्रॉस-जाँचता है;
- क्रॉस-कटिंग नियम: `owner` और सारे हैश/प्रतिबद्धताएँ
  **पुनर्गणित** होती हैं, किसी दिए गए मान से कभी नहीं ली जातीं।

## सहमति नियम

शिप किए गए (`/docs/en/technical/blockchain.md` में वर्णित): हस्ताक्षरित
ब्लॉक उत्पादक (`SCONE-BLOCK-V1` पेलोड, चेकपॉइंट-निर्वाचित
समिति), समुच्चित चेकपॉइंट द्वारा फ़ाइनलिटी (`SCONE-CKPT-V1`, फ़ाइनलिटी
फ़्लोर पर फ़ॉर्क चयन), नाम लंबाई के अनुसार रजिस्ट्रेशन PoW।
शेष विषय: सूक्ष्म एंटी-स्पैम शुल्क, स्वामी कुंजी रोटेशन।

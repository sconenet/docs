# आर्किटेक्चर

## पहले सरल शब्दों में

बाकी सब कुछ समझने के लिए पाँच अवधारणाएँ पर्याप्त हैं:

1. **एक नाम** — इंटरनेट पर एक पता, जैसे `mysite.scone`।
2. **आपका 24-शब्दों का वाक्यांश** — आपकी पहचान। यही वह दस्तावेज़ है जो
   साबित करता है कि एक नाम आपका है, और आपके द्वारा किए गए हर बदलाव पर हस्ताक्षर करता है।
3. **सार्वजनिक रिकॉर्ड पुस्तक** (ब्लॉकचेन) — किसके पास क्या है, इसकी साझा,
   सार्वजनिक बही। इसे कोई नियंत्रित नहीं करता, इसे कोई
   फिर से लिख नहीं सकता।
4. **माइनिंग** — नाम आरक्षित करने का तरीका: आपका कंप्यूटर कुछ
   कार्य करता है, सबके लिए निष्पक्ष और समान। कोई कतार नहीं, कोई बिचौलिया नहीं।
5. **एक रिले** — एक कंप्यूटर जो पुस्तक की एक प्रति रखकर और सवालों के
   जवाब देकर नेटवर्क को चलाता है। कोई भी इसे चला सकता है; जितने अधिक होंगे, नेटवर्क उतना ही मज़बूत होगा।

इस पृष्ठ का बाकी हिस्सा गहराई में जाता है — आप यहीं रुक सकते हैं और
फिर भी प्रोजेक्ट को समझ चुके होंगे।

## मार्गदर्शी सिद्धांत

सकोन प्राधिकार को उपलब्धता से अलग करता है:

- **ब्लॉकचेन** स्वामित्व और बदलावों के क्रम के लिए सत्य का स्रोत है;
- **DHT** एक वितरण/प्रतिकृति प्रणाली है, यह कोई **प्राधिकार नहीं** है;
- **स्थानीय भंडारण** नोड की अवस्था, अनुक्रमणिकाएँ और कैश रखता है।

एक नोड को DHT से आने वाले किसी भी डेटा को संभावित रूप से
दुर्भावनापूर्ण मानना चाहिए: वह केवल क्रिप्टोग्राफ़िक सत्यापन के बाद ही
स्वीकृत होता है (स्वामी का हस्ताक्षर, चेन के स्वामित्व / अनुक्रम /
record_hash से मिलान)।

## ज़िम्मेदारियों का पृथक्करण

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

## Crates और निर्भरताएँ

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

### निर्णय: `scone-core`, `scone-crypto` पर निर्भर है

`DomainId` को नींव से ही संगणनीय होना चाहिए
(`BLAKE3-256(prefix || canonical name)`), लेकिन ठोस क्रिप्टोग्राफ़ी को
कोर में दूषित नहीं होना चाहिए। समाधान:

- `scone-crypto` सबसे निचली परत है: बाइट्स पर विशुद्ध रूप से नियतिवादी फ़ंक्शन
  (कुंजी निर्माण के लिए OS RNG के अतिरिक्त), शून्य कोर प्रकार,
  कोई नेटवर्क/फ़ाइलतंत्र/async निर्भरता नहीं → कोई चक्रीयता संभव नहीं;
- `scone-core` इसका उपयोग केवल हैशिंग के लिए करता है;
- Ed25519 हस्ताक्षर और कुंजियाँ `scone-crypto` में रहती हैं और
  बाइट्स पर काम करती हैं (कैननिकल पेलोड `scone-protocol` द्वारा निर्मित)।

### प्रत्येक crate की भूमिका

**scone-core** — नाम (`TldName`, `DomainName`), पहचानकर्ता
(`DomainId`, `OwnerId`, `PublicKeyRef`), DNS डेटा (`DnsRecord`,
`SignedDnsRecord`), लेनदेन (`RegisterDomain`, `UpdateDomain`)।
नियतिवादी और पोर्टेबल। इस crate में निषिद्ध: libp2p, redb,
tokio/async, नेटवर्क, फ़ाइलें, OS, DNS सर्वर, ब्लॉकचेन तर्क।

**scone-crypto** — क्रिप्टोग्राफ़िक प्रिमिटिव। आज: `hash256`
(BLAKE3), Ed25519 हस्ताक्षर और कुंजियाँ (`SigningKey`,
`PublicKey`, `Signature`, सख्त बैच सत्यापन), रिकॉर्ड समूहों का
AEAD लिफ़ाफ़ा (XChaCha20-Poly1305)। योजनाबद्ध: मर्कल ट्री।
नियम: कोई स्वनिर्मित एल्गोरिद्म नहीं, केवल ऑडिट की गई निर्भरताएँ।

**scone-protocol** — प्रोटोकॉल का बाइनरी प्रारूप: लेनदेन, DNS
रिकॉर्ड, ब्लॉक (`Block`/`BlockHeader`, एक अपारदर्शी सहमति फ़ील्ड के साथ)
और P2P संदेशों का कैननिकल एनकोडिंग/डिकोडिंग (न्यूनतम LEB128 varint,
शुद्ध 32-बाइट पहचानकर्ता), आवंटन से पहले आकार सीमाओं की जाँच के साथ,
गैर-कैननिकल रूपों की अस्वीकृति, और संस्करणीकरण
(`PROTOCOL_VERSION`)। कोई सहमति, नेटवर्क या भंडारण तर्क नहीं।
प्रारूप संदर्भ: `/docs/en/technical/protocol.md`।

**scone-blockchain** — ब्लॉकचेन तर्क और कैननिकल अवस्था:
लेनदेन पहचान (`TxId`), क्रमबद्ध मर्कल ट्री,
`BlockHash`, नियतिवादी जेनेसिस, ब्लॉक और लेनदेन सत्यापन,
अवस्था अनुप्रयोग नियम (`DomainState`/`ChainState`, मेमोरी में,
`DomainId` द्वारा प्रत्यक्ष पहुँच), फ़ॉर्क पहचान और शाखा चयन
(`fork_choice`: फ़ाइनलिटी फ़्लोर + सबसे लंबी चेन + सबसे-कम-हैश
टाई-ब्रेक), चेकपॉइंट फ़ाइनलिटी, और सहमति अमूर्तन
(`trait Consensus`, डिफ़ॉल्ट रूप से अनुमेय कार्यान्वयन)।
रजिस्ट्रेशन PoW (mine/verify, प्रति-नेटवर्क कठिनाई) `scone-core`
में रहता है। इस crate में निषिद्ध: नेटवर्क, DHT, भंडारण (redb),
DNS सर्वर। संदर्भ: `/docs/en/technical/blockchain.md`।

**scone-storage** — स्थानीय स्थिरता (शिप किया गया)। ठोस बैकएंड
`redb` (`NodeStore`) है, जो एक न्यूनतम trait के पीछे छिपा है ताकि
ब्लॉकचेन/DHT/DNS कभी भी सीधे redb से युग्मित न हों; कोई दूसरा
बैकएंड (वितरित KV) प्लग किया जा सकता है। इसमें पेजिनेटेड
KV अवस्था बैकएंड, ब्लॉक लॉग, सत्यापित स्नैपशॉट (हैश किया मैनिफ़ेस्ट
+ पेज) और पुनरारंभ रिस्टोर/रीप्ले पथ शामिल हैं। संदर्भ:
`/docs/en/technical/storage.md`।

**scone-network** — नोड का p2p रिले: libp2p/QUIC स्वॉर्म,
लेनदेन और ब्लॉक रिले (व्यवस्थित स्थानीय सत्यापन), बाध्य
मेमपूल, बाध्य ब्लॉक सिंक (128 ब्लॉक/अनुरोध), `SignedDnsRecord` के लिए
Kademlia DHT, UDP+TCP DNS सर्वर (GeoDNS हमेशा चालू —
एम्बेडेड GeoIP तालिका, रिकर्सिव फ़ॉलबैक), एंकरों का
चेकपॉइंट/हार्टबीट लूप (CLI रिले से सीधे p2p QUIC बोलता है,
अब कोई स्थानीय RPC नहीं)। पहुँच और खोज: AutoNAT
v2 (reachable/NATed निर्णय), NATed नोड्स के लिए circuit-relay v2 + DCUtR,
mDNS (ज़ीरो-कॉन्फ़िग LAN खोज, devnet पर डिफ़ॉल्ट रूप से चालू)
और UPnP (राउटर द्वारा QUIC पोर्ट मैपिंग, ऑप्ट-इन)। `.env` फ़ाइल के माध्यम से विन्यास (देखें `.env.example`)।
संदर्भ: `/docs/en/technical/relay.md`।

**scone-keystore** — Ed25519 हस्ताक्षर कुंजियों का एन्क्रिप्टेड भंडारण
(`.sconekey` फ़ाइलें: Argon2id + XChaCha20-Poly1305)। यह crate
फ़ाइलतंत्र का स्वामी है ताकि `scone-crypto` शुद्ध रह सके। संदर्भ:
`/docs/en/technical/keystore.md`।

**scone-topology** — स्थानीय p2p टोपोलॉजी अनुकूलन के लिए शुद्ध इंजन:
शून्य नेटवर्क, शून्य फ़ाइलतंत्र, शून्य async। यह न तो libp2p को
जानता है न स्वॉर्म को: `scone-network` इसे अवलोकन देता है (ping RTT,
कनेक्शन) और इससे पड़ोस प्रबंधन निर्णय प्राप्त करता है।

**scone-genesis** — मेननेट जेनेसिस तैयारी के लिए वन-शॉट टूल बाइनरी
(किकस्टार्टर): संस्करणित JSON मैनिफ़ेस्ट, आवंटन,
नियतिवादी ब्लॉक।

**scone** — बाइनरी/CLI। किसी crate पर केवल तब निर्भर करता है जब वास्तव में
उसका उपयोग करता हो (आज: रिले के लिए scone-network, scone-blockchain,
scone-storage, scone-keystore, scone-protocol, scone-core,
scone-crypto, scone-topology)।

## नोड

एक सकोन नोड में संयुक्त रूप से शामिल हैं:

1. एक ब्लॉकचेन (सहमति सत्यापन, स्वामित्व अवस्था);
2. DHT रूटिंग (`SignedDnsRecord` का प्रकाशन/पुनर्प्राप्ति);
3. स्थानीय भंडारण (अवस्था, अनुक्रमणिकाएँ, कैश);
4. सामने एक रिज़ॉल्वर / DNS सर्वर।

### नया नोड शुरू करना (अवस्था बूटस्ट्रैप, P0.3/P0.4)

एक नया नोड जेनेसिस को दोबारा नहीं चलाता: वह एक सत्यापित
फ़ाइनल अवस्था से शुरू होता है।

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

दो नोड प्रोफ़ाइलें:

- **आर्काइव नोड** (ऑपरेटर फ़्लैग): सारे ऐतिहासिक ब्लॉक रखता है,
  उन्हें नेटवर्क को देता है, मौजूदा पथों से शुरू होता है;
- **फ़ुल-स्टेट नोड**: H पर पूरी अवस्था और प्रत्यय रखता है,
  लेकिन H से नीचे का इतिहास नहीं — यह भविष्य के ब्लॉकों को सत्यापित करता है और
  अवस्था देता है, यह इतिहास नहीं देता।

यह पथ ऑप्ट-इन है: सत्यापित स्नैपशॉट का आयात एक स्पष्ट
ऑपरेटर क्रिया है (`import_verified_snapshot`), कभी स्वचालित नहीं —
किसी मौजूदा नोड का डिफ़ॉल्ट व्यवहार नहीं बदलता।
नॉर्मेटिव विवरण: `/docs/en/technical/storage.md`, अनुभाग
"VERIFIED state snapshot" और "State bootstrap"।

## DNS रिज़ॉल्यूशन प्रवाह

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

DNS स्नैपशॉट (प्रत्येक जीवित डोमेन का अस्तित्व + समाप्ति,
प्रत्येक लागू ब्लॉक के बाद प्रकाशित) गारंटी देता है कि कोई अस्तित्व
निर्णय कभी भी रिले के रन लूप से नहीं गुज़रता: कोर को
केवल वास्तविक रिकॉर्ड प्राप्तियों के लिए छुआ जाता है।

## विकसनीयता

- संस्करणित प्रीइमेज (`SCONE-DOMAIN-V1`, `SCONE-PUBKEY-V1`,
  `SCONE-OWNER-V1`, …): किसी व्युत्पत्ति नियम को बदलने से मौजूदा
  नेमस्पेस टूटने के बजाय एक नया नेमस्पेस बनता है;
- `RecordData` वेरिएंट + `Unknown { type_code, data }` के माध्यम से
  विस्तारणीय, पढ़ने के समय अज्ञात प्रकारों के लिए;
- वायर प्रारूप टूटन के लिए `PROTOCOL_VERSION` (`scone-protocol`);
- चेन केवल हैश रखती है: बड़ा डेटा DHT में रहता है।

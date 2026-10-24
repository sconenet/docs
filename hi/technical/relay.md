# नेटवर्क रिले (M4) — आर्किटेक्चर, प्रवाह और CLI नियंत्रण

> सकोन नेटवर्क रिले (`scone-network`) का नॉर्मेटिव विनिर्देश।
> कोई भी doc/code विचलन = बग (सत्य के स्रोत को ठीक करें)।

## अवलोकन

**रिले** सकोन का पूर्ण P2P नोड है: यह कैननिकल
ब्लॉकचेन (RAM + `NodeStore` redb) रखता है, लेनदेन और ब्लॉक रिले करता
है, ब्लॉक उत्पादित करता है (devnet मोड), सिंक प्रोटोकॉल सेव करता है,
`SignedDnsRecord`s को Kademlia DHT में प्रकाशित और हल करता है, और
वह **p2p प्रोटोकॉल** (`/scone/reqres/1`) प्रदर्शित करता है जिससे CLI
सीधे प्रश्न करता है (RPC→p2p एकीकरण: JSON TCP चैनल गायब)।

```text
                    ┌────────────────────────────────────────┐
                    │              scone relay               │
  P2P (libp2p)      │                                        │
  QUIC (udp,        │  Swarm libp2p                          │
  ephemeral port    │  ├─ identify  (address discovery)      │
  by default) ◄─────┤  ├─ ping      (liveness)               │
                    │  ├─ kad       (record DHT)             │
                    │  └─ reqres    (scone-prot. messages)   │
                    │                                        │
                    │  Relay core (single task)              │
                    │  ├─ Blockchain (full validation)       │
                    │  ├─ Mempool (bounded, dedup by TxId)   │
                    │  ├─ BlockBuilder (devnet production)   │
                    │  ├─ Fork parking (bounded, 2048)       │
                    │  └─ Sync driver (batched catch-up)     │
                    │                                        │
  CLI ◄─────────────┤  reqres p2p (QUIC, same messages)      │
                    └────────────────┬───────────────────────┘
                                     │
                              NodeStore (redb)
```

`crates/scone-network` workspace की **एकमात्र async crate** है:
उसके नीचे सब कुछ (crypto → core → protocol → blockchain →
storage) समकालिक और शुद्ध रहता है।

## कनेक्शन प्रवेश (P1.5)

स्वॉर्म स्तर पर कठोर सीमाएँ (libp2p `connection_limits` व्यवहार,
`SconeBehaviour` की पहली प्रविष्टि):

- `MAX_TOTAL_CONNECTIONS = 128` — प्रति रिले स्थापित कनेक्शनों
  (incoming + outgoing) पर सीमा; अधिक incoming REFUSED,
  कभी कतार में नहीं;
- `MAX_PER_PEER_CONNECTIONS = 4` — एक अकेला पीयर कई
  कनेक्शन खोलकर कुल कोटा पर एकाधिकार नहीं कर सकता;
- `MAX_PENDING_INCOMING = 32` — slowloris रक्षा: अप्रमाणित
  अर्ध-कनेक्शन कभी अनिश्चित रूप से प्रतीक्षा नहीं करते।

ये सीमाएँ मौजूदा प्रति-संदेश सीमाओं
(`MAX_MESSAGE_LEN` 1 MiB) और प्रति-बजट सीमाओं (sync `max ≤ 128`
ब्लॉक, DNS TCP 64 कनेक्शन/512 B/5 s) के पूरक हैं। कनेक्शन-बाढ़
हमलावर को इनकार मिलते हैं, मेमोरी वृद्धि नहीं।

## पीयर खोज

तीन पूरक तंत्र, इस प्राथमिकता क्रम में:

1. **स्पष्ट seeds** (`--bootstrap`, पूर्ण multiaddrs) — सदा
   प्राथमिकता लेते हैं; ग़ैर-रिक्त सूची देना बाकी सब को बदल देता है।
   `--no-builtin-seeds` 100% निजी परिनियोजन के लिए एम्बेडेड seeds
   खाली करता है।
2. **एम्बेडेड seeds** (`scone-network/src/config.rs` में
   `builtin_seeds(network)`) — प्रत्येक आधिकारिक बाइनरी
   रिलीज़ पर जमे, प्रति नेटवर्क। बिना विन्यस्त रिले नेटवर्क से
   अकेला जुड़ता है। (सार्वजनिक testnet तैनात होने तक testnet पर
   खाली।)
3. **सीखी गई निर्देशिका** (`<data-dir>/peers.json`) — `identify`
   से सीखा प्रत्येक पता रखा जाता है (अंतिम दर्शन; **2000
   प्रविष्टियों** की सीमा — पते, कनेक्शन नहीं, अधिकतम पर ~400 KiB
   RAM — और **90-दिन आयु कटौती**: 90 दिन से पुरानी प्रविष्टी लेखन
   समय पर गिराई और लोड समय पर उपेक्षित जाती है, जिससे 2000 स्लॉट
   संभावित पीयरों से भरे रहते हैं बजाय प्री-चर्न जीवाश्मों के जिनका
   एकमात्र योगदान बूट-समय डायल टाइमआउट होता) और उत्पादन
   tick पर ~1×/मिनट सहेजी। बिना seed और बिना bootstrap वाले बूट पर
   निर्देशिका से पुनर्गठित: रिले बिना
   विन्यास अपने सामान्य पीयर ढूँढ लेता है। यह कैश है: गायब/भ्रष्ट
   फ़ाइल = खाली निर्देशिका, कभी त्रुटि नहीं।
4. **UPnP** (`--upnp`, ऑप्ट-इन) — होम राउटर द्वारा QUIC
   सुनने-पोर्ट का मैपिंग: नोड बिना
   circuit relay के बाहर से पहुँच योग्य हो जाता है। `NewExternalAddr`
   पर, बाहरी पता स्वॉर्म के वास्तविक बाहरी पते के रूप में पंजीकृत
   (`add_external_address` — identify/autonat द्वारा देखा, एक
   transport-relay उम्मीदवार)। कोई IGD गेटवे नहीं (`GatewayNotFound`)
   या ग़ैर-रूट करने योग्य गेटवे (`NonRoutableGateway`): सादा info
   लॉग, मैपिंग छोड़ दिया — होम LAN पर सामान्य स्थितियाँ। समाप्त
   मैपिंग (नवीनीकरण विफल) चेतावनी है।

UPnP **डिफ़ॉल्ट रूप से अक्षम** रहता है
(`config.rs` में `enable_upnp` = `false`): सार्वजनिक परिनियोजन
सहमति के बिना किसी राउटर से बात नहीं करता। **mDNS खोज हटा दी
गई** (remove-mdns-discovery, 2026-10-22): peering केवल स्पष्ट
bootstrap, बिल्ट-इन seeds और Kademlia से आता है — पुराना
`--mdns`/`SCONE_MDNS` flag बिना प्रभाव स्वीकार किया जाता है
(स्क्रिप्ट संगतता, `--rpc` जैसा ही व्यवहार)।

### स्थानीय टोपोलॉजी अनुकूलन (हमेशा चालू)

रिले शुद्ध `scone-topology` इंजन (crate दस्तावेज़ देखें) से अपना
पड़ोस सक्रिय रूप से अनुकूलित करता है। **हमेशा सक्षम**;
एकमात्र उजागर ट्यूनिंग एक **profile** है (`--topology-profile`,
env `SCONE_TOPOLOGY_PROFILE`, डिफ़ॉल्ट `default`):

| Profile | min / target / max | मू. | किसके लिए |
|---|---|---|---|
| `tiny` | 2 / 4 / 6 | 120 s | Raspberry Pi, होम बॉक्स, 1-vCPU VPS |
| `small` | 4 / 8 / 12 | 90 s | छोटा सर्वर, होम NAS |
| `default` | 8 / 16 / 32 | 60 s | मानक सर्वर (डिफ़ॉल्ट) |
| `high` | 16 / 32 / 64 | 45 s | संप्रभु नोड, बैकबोन |
| `very-high` | 32 / 64 / 128 | 30 s | संस्थापक हब |
| `devtest` | 2 / 4 / 6 | 5 s | **केवल LAB/TEST** (डिज़ाइन से अस्थिर) |

```
scone relay --topology-profile tiny
```

सुरक्षात्मक पैरामीटर (hysteresis 0.15, cooldown 600 s, grace
120 s, माप 15 s) हर जगह समान हैं: वे स्थिरता
रखते हैं और नोड आकार के साथ बढ़ने का कारण नहीं हैं। अज्ञात
नाम टाइप्ड startup त्रुटि है।

असंगत विन्यास (`min > target > max`, सीमा के बाहर
threshold `[0,1]` — `NaN` सहित —, शून्य अंतराल) **टाइप्ड startup
त्रुटि** है, कभी मौन व्यवहार नहीं।

**दृश्य (`TopologyView`)** — स्वॉर्म घटनाओं से पोषित:

- सफल `ping` → RTT अवलोकन (8 नमूनों की माध्य खिड़की);
- विफल `ping` → विश्वसनीयता दंड;
- `identify` प्राप्त → पते (≤ 4/पीयर) + घोषित प्रोटोकॉल
  (kad/reqres/identify/relay bitmask);
- कनेक्शन स्थापित → जुड़ी अवस्था, दिशा
  (incoming/outgoing), आयु, पहचाना circuit-relay;
- कनेक्शन बंद → दंड + churn काउंटर केवल तब जब अनचाहा हो
  (स्वैच्छिक निष्कासन गिना नहीं जाता);
- outgoing dial विफलता → विश्वसनीयता दंड।

दृश्य सीमित है (500 पीयर; उसके आगे केवल जुड़े पीयर
याद रखे जाते हैं)।

**kbuckets से `kad_required`** — प्रत्येक मूल्यांकन से पहले,
रिले Kademlia रूटिंग तालिका चलता है: पीयर सुरक्षित है यदि वह किसी
बकेट में `Connected` हो, या SATURATED बकेट में `Disconnected`
(k = 20)। सुरक्षित पीयर निष्कासन के लिए कभी **नहीं चुना जाता**।

**अनुकूलन लूप** — profile के मूल्यांकन अंतराल पर
(`default` में 60 s; ऊपर की तालिका देखें):

1. `kad_required` पुनर्गणना (kbuckets);
2. इंजन मूल्यांकन (hysteresis: 120 s अनुग्रह अवधि, न्यूनतम स्कोर
   सुधार +15%, निष्कासन cooldown 600 s);
3. निर्णय लागू: `AddCandidate` → उम्मीदवार dial करें;
   `Replace` → निष्कासित पीयर बंद करें (एक बार — प्रगतिशील, कोई
   लूपिंग नहीं) फिर प्रतिस्थापन dial करें;
4. एक **metrics** स्नैपशॉट लॉग होता है (INFO स्तर):
   known/connected/incoming/outgoing/circuit/protected गिनती,
   माध्य/माध्यिका RTT, विशिष्ट IP बकेट, अधिकतम सांद्रता, churn
   (प्रतिस्थापन/विच्छेदन)।

पड़ोस स्कोर कभी विश्वास माप नहीं है (`scone-topology`
दस्तावेज़ देखें): इससे कोई सहमति निर्णय व्युत्पन्न नहीं होता।

### पहुँच (AutoNAT) और परिवहन रिले (circuit-relay v2)

प्रत्येक रिले AutoNAT (क्लाइंट + सर्वर) चलाता है: नोड अपने
पीयरों से अपनी ओर वापस dial का प्रयास करवाता है; निर्णय
(`reachable`: हाँ = बाहर से पहुँच योग्य, नहीं = NAT के पीछे,
अज्ञात = अभी परीक्षित नहीं) हर 5 मिनट ताज़ा और `NodeStatus` में
प्रदर्शित (फ़ील्ड `nat_reachable`, वायर varint 0/1/2) — CLI दिखाता
है (`scone status`)। निजी पते गिने जाते हैं
(`only_global_ips = false`): एक LAN मेश अपनी LAN पहुँच
परीक्षण करता है।

पहुँच योग्य रिले circuit-relay v2 **hop** सक्षम करता है (यह
NATed पीयरों के लिए ट्रैफ़िक रिले कर सकता है) और **DCUtR**
(hole-punching: circuit से जुड़े दो नोड सीधा कनेक्शन आज़माकर
रिले मुक्त करते हैं)। NATed माना गया रिले स्वचालित रूप से
पहले ज्ञात पहुँच योग्य पीयर से आरक्षण माँगता है और उसके द्वारा
सुनता है (पता `/p2p/<relay>/p2p-circuit`)।

उत्पादन नियम: रिले होकर NATed नोड ब्लॉक
उत्पादक नहीं बनना चाहिए (घटी प्रतिक्रिया/उपलब्धता) — PoS नियम
(सीट = जीवित डोमेन) इसे अभी मना नहीं करता; सार्वजनिक testnet से
पहले संचालन आवश्यकता के रूप में प्रलेखित होना है।

## परिवहन और libp2p प्रोटोकॉल

| प्रोटोकॉल | नाम | भूमिका |
|---|---|---|
| QUIC (quic-v1) | — | मुख्य परिवहन (udp) |
| identify | `/scone/id/1` | पीयर पता खोज (Kademlia तालिका भरता है) |
| ping | libp2p मानक | जीविता |
| Kademlia | `/scone/kad/1` | DHT: कुंजी = `DomainId` (32 B), मान = `SignedDnsRecord` की कैननिकल एनकोडिंग |
| request-response | `/scone/reqres/1` | scone-protocol अनुप्रयोग संदेश |

### अनुप्रयोग फ़्रेम (`/scone/reqres/1`)

प्रत्येक फ़्रेम `len: u32 BE || bytes` है जहाँ बाइट्स
`scone-protocol` के `Message` की कैननिकल एनकोडिंग हैं (मौजूदा
कोडेक — कोई दूसरा प्रारूप नहीं)। लंबाई किसी भी पढ़ या आवंटन से
**पहले** `limits::MAX_MESSAGE_LEN` से सीमित: बड़ा फ़्रेम घोषित
करने वाला पीयर शून्य मेमोरी लागत पर अस्वीकृत।

विनिमय संदेश (प्रति request/response round-trip एक):

| संदेश | दिशा | उत्तर |
|---|---|---|
| `Hello { version }` | handshake | `Hello` (स्थानीय संस्करण) |
| `Ping` / `Pong` | जीविता | `Pong` |
| `GetBlock { hash }` | प्रश्न | `Block` (RAM खिड़की) या अज्ञात हो तो `Pong` |
| `GetBlocks { start, max }` | sync | **एक** `Block` (ऊँचाई `start`), या कुछ न हो तो `Pong` |
| `Block (b)` | ब्लॉक रिले | `Pong` (ack) |
| `Transaction (tx)` | tx रिले | `Pong` (ack) |
| `GetRecord { domain_id }` | स्थानीय DHT | `Record` (स्थानीय कैश) या `Pong` |
| `Record (r)` | प्रकाशन | स्वीकृत हो तो `Pong`, अन्यथा त्रुटि |

नियम: **प्रति अनुरोध ठीक एक उत्तर**; प्राप्ति-सूचना संदेश
`Pong` हैं। Sync उत्तर एक समय एक ब्लॉक लाते हैं — पूर्ति
अनुरोधकर्ता पक्ष दोहराता है ("Sync" देखें)। यह जानबूझकर
सीमित है: प्रति पीयर उड़ान में कभी एक से अधिक ब्लॉक नहीं।

## रिले व्यवहार

### स्टार्टअप

0. नेटवर्क रिज़ॉल्यूशन (`--network testnet|mainnet|devnet`, प्रोजेक्ट
   dev में होने तक डिफ़ॉल्ट testnet — M8b)। Data-dir **प्रति
   नेटवर्क** (`~/.scone/<network>/`): ग़ैर-रिक्त store जो दूसरे
   नेटवर्क की चेन रखता हो इनकार (`Corrupted`);
1. redb store खोलें/बनाएँ (`<data-dir>/chain.redb`);
2. `load_chain(store, network)`: O(1) tip + पेजिनेटेड डोमेन अवस्थाएँ
   (RAM खिड़की: जेनेसिस + tip);
3. स्वॉर्म निर्माण (प्रति-सत्र अल्पकालिक Ed25519 पहचान — नेटवर्क
   पहचान स्थिर करना खुला निर्णय);
4. बूट सूची बनाएँ (देखें
   [पीयर खोज](#पीयर-खोज)): स्पष्ट `--bootstrap`
   (`/p2p/<peer-id>` पर समाप्त होना चाहिए) अन्यथा नेटवर्क के
   एम्बेडेड seeds अन्यथा सीखी निर्देशिका (`peers.json`);
5. प्रत्येक multiaddr dial कर Kademlia तालिका में पंजीकृत करें
   (मृत dial उपेक्षित — बाकी नेटवर्क बनाते हैं)।

### लेनदेन प्राप्त करना (P2P या CLI जमा)

1. सख्त सीमित डिकोडिंग (`MAX_MESSAGE_LEN`);
2. पूर्ण क्रिप्टोग्राफ़िक सत्यापन (`validate_transaction`:
   owner/key बंधन, कैननिकल पेलोड पर `verify_strict`);
3. **पहले एंटी-रीप्ले** (सस्ता, अवस्था-स्वतंत्र, निश्चित),
   फिर अवस्था पूर्व-जाँच (`push_block` नियमों का सस्ता दर्पण —
   PoW, जो महँगा है, केवल push-समय पर सत्यापित): रीप्ले किया
   `RenewDomain` को `TxReplay` मिलता है, झूठी PoW त्रुटि नहीं
   (उसका प्रमाण वह समाप्ति बाँधता है जिसे पहला अनुप्रयोग पार कर
   चुका)। नेटवर्क
   पृथक्करण चरण 2 पर लागू: हस्ताक्षर पेलोड रिले का
   नेटवर्क इंजेक्ट करता है, कहीं और हस्ताक्षरित tx
   `InvalidSignature` से विफल (वायर 2026-09: अब कोई `network`
   फ़ील्ड नहीं);
   `RegisterDomain` → पंजीकृत नाम का TLD
   (अन्यथा `UnknownTld`, D1/M7c) **और खुला** (अन्यथा
   `TldClosed`, M8b) फिर नाम खाली; `UpdateDomain` → ज्ञात डोमेन,
   सटीक अनुक्रम `current + 1`, स्वामी; `RegisterTld` → खाली TLD;
   `TransferTld`/`RevokeTld`/`SetTldOpen` → ज्ञात TLD + स्वामी
   हस्ताक्षरकर्ता; `AssignDomain` → TLD स्वामी + खाली नाम;
   `RenewTld` → ज्ञात निजी TLD + स्वामी, निश्चित अवधि पुनर्व्युत्पन्न, अनुग्रह-विंडो पूर्वजाँच; `RenewDomain` → ज्ञात डोमेन + स्वामी, नवीनीकरण PoW पूर्वजाँच
   (वर्तमान समाप्ति से बंधी) अगली ऊँचाई पर;
4. मेमपूल प्रविष्टि (निश्चित क्षमता, `TxId` से dedup —
   पूल में मौजूद tx दोबारा जमा करना no-op);
5. पीयरों को प्रसारण (प्रेषक को छोड़कर)।

### ब्लॉक प्राप्त करना (P2P, sync, या स्व-उत्पादित)

1. `push_block`: मौजूदा पूर्ण सत्यापन (जनक = कैननिकल tip,
   ऊँचाई, पुनर्गणित Merkle रूट, प्रत्येक लेनदेन सत्यापित फिर
   लागू — प्रति ब्लॉक परमाणु);
2. स्वीकृत होने पर: रिले अनुप्रयोग परिणाम पढ़ता है
   (`push_block_with_gc` → हैश + नियतिवादी GC द्वारा हटाए
   डोमेन) और `store_block_with_removals` से स्थिर करता है
   (परमाणु डेल्टा जोड़: ब्लॉक + tip + बदली अवस्थाएँ — upserts
   और हटाने दोनों: समाप्त डोमेन, निरस्त TLDs)। पुनर्चालन इसलिए
   कभी समाप्त पंजीकरण पुनर्जीवित नहीं कर सकता;
3. मेमपूल समायोजन (सम्मिलित txs हटाए);
4. पीयरों को प्रसारण (प्रेषक को छोड़कर)।

अमान्य ब्लॉक टाइप्ड त्रुटि से अस्वीकृत; रिले चलता
रहता है। नेटवर्क डेटा पर कोई panic नहीं। भारी लागू
(Argon2 + redb commit) `tokio::task::block_in_place` से जाते
(sync driver, p2p accept, उत्पादन): runtime तैयार tasks को दूसरे
workers पर हटाता है — ब्लॉक लागू करना अब DNS सेवा या
metrics को रोकता नहीं।

### DNS स्नैपशॉट (प्रकाशन)

प्रत्येक लागू ब्लॉक के बाद (p2p accept — reorg सहित —, sync
driver, उत्पादन) और बूट पर (पुनर्चालन अगले ब्लॉक तक जीवित
नामों को NXDOMAIN न करे), रिले DNS सतह द्वारा पढ़े `watch`
feed पर **प्राधिकारी DNS स्नैपशॉट** प्रकाशित करता है (पूर्ण
विनिर्देश `dns.md` में):

- tip विस्तार (सर्वाधिक सामान्य पथ): **O(touched) डेल्टा**,
  copy-on-write — ब्लॉक के लेनदेन छुए डोमेनों का नाम लेते
  हैं, साथ ही नियतिवादी GC हटान
  (`AppliedBlock.gc_removed_domains`, push-समय पर मूल्यांकित);
- reorg / बूट: **पूर्ण पुनर्निर्माण** (एक पेजिनेटेड
  `ChainState::dns_facts` walk)।

### ब्लॉक उत्पादन (M5: हस्ताक्षरित उत्पादक)

प्रत्येक `produce_interval` सेकंड (डिफ़ॉल्ट 2 s): यदि मेमपूल
ग़ैर-रिक्त है, `BlockBuilder` द्वारा ब्लॉक जोड़ा जाता है
(`timestamp = now`) **अधिकृत उत्पादक द्वारा हस्ताक्षरित** (पेलोड
`header.consensus` में `SCONE-BLOCK-V1`, देखें
`/docs/en/technical/blockchain.md`) और प्राप्त ब्लॉक के
स्वीकृति पथ का ठीक अनुसरण करता है। बिना tx वाला रिले कुछ
उत्पादित नहीं करता।

उत्पादन कुंजी: एंकर कुंजी (`--anchor-key`) जब रिले
उससे सशस्त्र हो — एंकर एक अधिकृत उत्पादक है — अन्यथा
निश्चित devnet कुंजी, वैध जब तक उत्पादक पूल खाली
(खुला बूटस्ट्रैप: जेनेसिस)। जैसे ही कोई जीवित डोमेन/TLD अस्तित्व
में आता है, devnet कुंजी द्वारा हस्ताक्षरित ब्लॉक अस्वीकृत
(`InvalidProducer`)।

### एंकर लूप (फ़ाइनलिटी, M5)

`--anchor-key` के साथ (`.sconekey` कीफ़ाइल, पासफ़्रेज़
`--anchor-passphrase-env` से, डिफ़ॉल्ट `SCONE_ANCHOR_PASS` —
`scone-network` लाइब्रेरी डिफ़ॉल्ट `SCONE_ANCHOR_PASSPHRASE` तक
बाइनरी से कभी नहीं पहुँचता, clap सदा बदल देता है):

- **प्रस्तावक रोटेशन**: ठीक एक नामित समिति सदस्य प्रत्येक
  epoch में स्व-प्रस्ताव देता है (`committee[epoch % size]`,
  नियतिवादी); बाकी सदस्य कभी अपनी टिप हस्ताक्षरित नहीं करते —
  चेकपॉइंट की सामग्री प्रस्तावक की टिप पर निर्भर है, और बिना
  गेट की स्व-प्रस्तावनाएँ epoch को प्रतिद्वंद्वी सामग्रियों में
  चीर देती थीं जिन्हें हर रक्षक ठीक ही इक्विवोकेशन के रूप में
  अस्वीकार करता (testnet, 2026-10-21);
- प्रत्येक स्वीकृत/उत्पादित ब्लॉक के बाद, यदि यह नोड वर्तमान
  समिति का सदस्य है **और** समिति
  `MIN_FINALITY_COMMITTEE_SIZE` छूती है, रिले अपने
  `SignerGuard` से प्रस्तावित चेकपॉइंट हस्ताक्षरित करता है
  (data-dir में `signer-state.dat` — एक कुंजी = एक वोट/epoch,
  crash-safe fail-closed) और उसे gossip करता है;
- **प्राप्ति पर वोट**: ऐसा प्रस्ताव जिसकी सामग्री हमारी चेन पर
  वैध हो (संदर्भित ब्लॉक मौजूद, उसी हैश पर) और समिति सदस्य के
  हस्ताक्षर से सुसज्जित हो, हमारा हस्ताक्षर भी पाता है — उसके
  बिना हर एंकर केवल अपनी टिप की सामग्री हस्ताक्षरित करता और
  कोई कोरम कभी नहीं बनता था;
- प्राप्त हस्ताक्षर (`Message::Checkpoint`) `CheckpointData`
  हैश से संचित (256 प्रविष्टियों तक सीमित); एक ही
  डेटा पर `quorum_for(समिति आकार)` सदस्य हस्ताक्षर →
  `accept_checkpoint` + फ़ाइनल चेकपॉइंट का प्रसारण;
- **फ़ाइनलिटी पर्सिस्टेंस**: चेकपॉइंट विंडू और जमा बेस हर
  फ़ाइनलाइज़ेशन पर स्टोर में लिखे और बूट पर पुनः लोड जाते
  हैं। चेकपॉइंट gossip अवस्था हैं (कभी ब्लॉक नहीं लाते) — इस
  पर्सिस्टेंस के बिना रीस्टार्ट खाली बेस से शुरू होता और
  पीयर-सेव विंडू दोबारा कभी श्रृंखलित नहीं कर सकता था
  (नेटवर्क की विंडू के नीचे के epoch एंकर गायब);
- **फ़ाइनलिटी कैच-अप**: जिस रिले का बेस खाली हो (नए नोड का
  बूट) हर tick पर पीयर से `GetCheckpoints` माँगता है जब तक
  बेस बहाल न हो जाए;
- बिना एंकर कुंजी वाला रिले **निष्क्रिय** रहता है: संचित
  करता है, फ़ाइनल करता है, रिले करता है, पर कभी हस्ताक्षर नहीं;
- `GetCheckpoints` → `checkpoint_window()` विंडू सेव;
- लूप की कोई भी त्रुटि लॉग और निगली जाती है — कभी घातक नहीं;
- स्थिति (`GetStatus`) `checkpoints` और `finalized_epoch` प्रदर्शित
  करती है।

### फ़ॉर्क पार्किंग और reorgs (sticky-minority सुधार)

एक ही ऊँचाई का बहन ब्लॉक, या जिसका जनक वर्तमान tip नहीं
(`ParentNotTip`/`UnknownParent`), कभी खोया नहीं जाता:
वह **पार्क** होता है (सीमित map, `PARKED_CAP = 2048` ब्लॉक, प्रत्येक
~KiB — बदतम स्थिति RAM सीमित; भरा पार्किंग नए
आने वालों को मौन से उपेक्षित करता है)। `accept_block` को
पहले दिए गए हैश dedup होते हैं (`seen_blocks`): दोबारा
gossip किया हारा ब्लॉक पुनर्मूल्यांकित नहीं होता।

प्रत्येक `Produce` tick पर, `reconsider_parked`:

1. पार्क किए ब्लॉकों का रैखिक निकास जिनका जनक tip बन गया;
2. यदि पार्क की शाखा वर्तमान tip से मज़बूत (fork-choice
   नियम अपरिवर्तित: फ़ाइनलिटी फ़्लोर, सबसे लंबी चेन, सबसे कम
   हैश), अपनाना — विस्थापित प्रत्यय स्वयं पार्क होता है (फ़ॉर्क के
   दोनों पक्ष मूल्यांकन योग्य रहते हैं)।

यह उस नोड को अटकने से बचाता है जिसने बराबर-ऊँचाई फ़ॉर्क का
हारने वाला पक्ष अपनाया: जीता ब्लॉक, एक बार दिया फिर इनकारा,
नियमित रूप से पुनर्मूल्यांकित होता है बजाय हमेशा के लिए भूले
जाने के (खोज chaos-30: 15 ms के अंतर पर दो बहन ब्लॉक)।

**Reorg स्थिरता**: दूसरी शाखा अपनाना एकदिष्ट जोड़
(`NonMonotonicHeight`) से नहीं जा सकता। रिले
`rewind_to_height(फ़ॉर्क बिंदु)` फिर `persist_state_snapshot`
(चेन के निर्यात से जीवंत अवस्था का पुनर्लेखन) फिर जीते
प्रत्यय का एकदिष्ट पुनः-जोड़ लागू करता है (`storage.md` देखें)।
स्थिरता विफलता लॉग और अवक्रमित (बूट पर रीप्ले से ठीक) — कभी
घातक नहीं।

## Sync (पूर्ति)

प्रत्येक पीयर कनेक्शन पर, रिले
`GetBlocks(start = local_height + 1)` माँगता है। प्रत्येक प्राप्त
`Block` जो अपेक्षित ऊँचाई ठीक पूरी करे, अगली ऊँचाई का
अनुरोध ट्रिगर करता है, तक कि समाप्ति (`Pong` = कुछ और नहीं) या
`MAX_SYNC_ROUNDS` (10,000) बैच — झूठे पीयर के विरुद्ध भी
पूर्ण सीमा। प्राप्त ब्लॉक पूर्ण सत्यापन से गुज़रते हैं: दुर्भावनापूर्ण
पीयर कुछ भी अमान्य स्वीकृत नहीं करवा सकता।

### रिकॉर्ड DHT

- **put** (API/web प्रवाह — CLI हेक्स चैनल हटाया गया): सख्त सीमित
  डिकोडिंग (`MAX_DHT_CACHE_ENTRY`), ऑन-चेन अवस्था से सत्यापन
  (owner, sequence, पुनर्गणित `record_hash` — "रिकॉर्ड
  सत्यापन" देखें), फिर `kad.put_record` (स्थानीय भंडारण +
  निकटतम पीयरों को प्रतिकृति, प्रतिकृति गुणक का
  `Quorum::Majority`) और स्थायी redb कैश।
- **get** (`domain resolve`): `kad.get_record`; waiter अपने
  `QueryId` और माँगे `DomainId` से बंधा — उत्तर केवल मेल
  खाती क्वेरी हल करता है, और रिकॉर्ड कुंजी ठीक
  माँगा डोमेन होना चाहिए (ऑन-चेन सत्यापन से **पहले** जाँचा)।
  परिणाम फिर लौटाने से पहले चेन से सत्यापित। समवर्ती
  अनुरोधों पर सीमा: `MAX_DHT_WAITERS` (256);
  उसके आगे तुरंत त्रुटि। सफलतापूर्वक सत्यापित रिज़ॉल्यूशन
  स्थायी कैश में लिखा जाता है (M5): स्थानीय पढ़
  (`domain_info`) फिर बिना नई DHT क्वेरी सेव करते हैं।
- Kademlia सर्वर मोड में चलता है, बिना रिकॉर्ड TTL (चेन
  वैधता शासित करती है, समय नहीं)।

### रिकॉर्ड सत्यापन ("चेन ही प्राधिकार है" नियम)

हल किया या प्राप्त `SignedDnsRecord` केवल तभी स्वीकृत/लौटाया
जाता है जब:

1. डोमेन ऑन-चेन पंजीकृत हो;
2. `record.owner` = वर्तमान ऑन-चेन स्वामी;
3. `record.sequence` = वर्तमान ऑन-चेन अनुक्रम;
4. `record_hash(record.record)` (पुनर्गणित BLAKE3) =
   ऑन-चेन कमिटेड `record_hash`।

अन्यथा: P2P पक्ष पर मौन अस्वीकृति, CLI पक्ष पर स्पष्ट त्रुटि।
रिकॉर्ड का Ed25519 हस्ताक्षर रिले द्वारा **प्राप्ति पर सत्यापित**
होता है (`ResolveLocal`: स्वामी की सार्वजनिक कुंजी
ऑन-चेन ज्ञात, हस्ताक्षर खुलना चाहिए — दूसरी
कुंजी द्वारा हस्ताक्षरित रिकॉर्ड अस्वीकृत, `record rejected`);
वह भविष्य सत्यापकों के लिए परिवहित भी होता है — ऑन-चेन
प्रतिबद्धता प्राथमिक
अखंडता नियंत्रण बनी रहती है।

## Prometheus मेट्रिक्स

एक HTTP endpoint (Prometheus text प्रारूप 0.0.4, हाथ से
घुमाया exposition — कोई नई निर्भरता नहीं) समर्पित task द्वारा
सेव:

- **डिफ़ॉल्ट रूप से बंद**: `--metrics-addr` के बिना कोई HTTP सर्वर नहीं चलता (मेट्रिक्स ऑप्ट-इन हैं)।
- `--metrics-addr ADDR:PORT`: उस पते पर एंडपॉइंट बाइंड करता है (जैसे स्थानीय `127.0.0.1:9101`, बाहर `0.0.0.0:9101`)।
- `--metrics-token TOKEN`: **निजी** मोड में बदलता है —
  `Authorization: Bearer *** आवश्यक (constant-time
  तुलना, अन्यथा `401`; token कभी लॉग या
  exposition में नहीं दिखता)।
- bind विफलता **endpoint अक्षम** करती है (ग़ैर-घातक चेतावनी) —
  कभी नोड नहीं।

प्रभावी bind stdout पर घोषित (machine-readable
अनुबंध, `p2p:` जैसा): सुनते ही `metrics: <addr>` पंक्ति।

**19 काउंटर** (`counter`, एकदिष्ट — हॉट पथ:
साझा `Arc<Metrics>` पर `fetch_add(Relaxed)`, कभी
सहमति इनपुट नहीं):

| समूह | श्रृंखलाएँ |
|---|---|
| Chain | `scone_blocks_applied_total`, `scone_blocks_rejected_total`, `scone_checkpoints_signed_total`, `scone_checkpoint_sigs_received_total` |
| Mempool | `scone_txs_admitted_total`, `scone_txs_rejected_total`, `scone_txs_evicted_grace_total`, `scone_txs_included_total` |
| DNS | `scone_dns_queries_total`, `scone_dns_cache_hits_total`, `scone_dns_cache_misses_total`, `scone_dns_local_chain_total`, `scone_dns_dht_answers_total`, `scone_dns_rejected_unverified_total` |
| DHT | `scone_dht_pulls_issued_total`, `scone_dht_negative_hits_total`, `scone_dht_negative_expired_total`, `scone_dht_negative_capped_total`, `scone_dht_cache_evictions_total` |
| Peer network | `scone_peer_connections_established_total`, `scone_peer_connections_closed_total`, `scone_peer_connections_incoming_total`, `scone_peer_dial_failures_total`, `scone_peer_blocks_received_total`, `scone_peer_transactions_received_total` |

**6 gauges** (`scone_height`, `scone_mempool`, `scone_peers`,
`scone_domains`, `scone_finalized_epoch`, `scone_addr_book_entries`)
— पहले पाँच हर 15 s पर `RelayCommand` चैनल से ताज़े
(CLI `status` जैसा ही पथ — सत्य का एकमात्र स्रोत);
`scone_addr_book_entries` रजिस्ट्री द्वारा सीधे अनुरक्षित
(प्रत्येक address-book निवेश पर अद्यतन, सत्य का स्रोत)।

E2E परीक्षण: `metrics_endpoint_public_and_private` (200 +
exposition, बिना/ग़लत token 401, Bearer के साथ 200)।

## CLI नियंत्रण (RPC→p2p एकीकरण)

JSON TCP RPC सर्वर **हटा दिया गया**: CLI (और रजिस्ट्रार का
axum पुल) p2p प्रोटोकॉल **सीधे** बोलते हैं — वही वायर
(`/scone/reqres/1`), रिले पक्ष पर वही handlers,
शून्य समानांतर चैनल। प्रत्येक कमांड नोड के multiaddr (`--peer`) पर
एक QUIC कनेक्शन खोलती, एक `Message` विनिमय करती, बंद करती।

रिले द्वारा सेव नियंत्रण संदेश:

- `GetStatus` → `NodeStatus { network, nat_reachable, tip,
  height, domain_count, mempool, peers, finalized_epoch,
  checkpoints, fresh_operators }` (`scone status` द्वारा प्रदर्शित —
  `fresh_operators` = ताज़ा हार्टबीट वाले ऑपरेटर, RFC
  proof-of-service);
- `LookupName { name }` → `NameState` (ऑन-चेन अवस्था, जैसा
  `lookup`);
- `GetOwnedNames { owner }` → `OwnedNames` (एक `OwnerId` के
  जीवित डोमेन + TLDs);
- `Transaction (tx)` → मुफ़्त gossip के साथ जमा (रिले
  पीयरों को पुनःप्रसारित करता है);
- `GetRecord { domain_id }` / `Record (r)` — DHT रिज़ॉल्यूशन और
  प्रकाशन (CLI पक्ष पर `scone domain resolve` जैसा);
- अनुप्रयोग अस्वीकृति → `Message::Error(msg)` (क्लाइंट तक
  पठनीय संदेश, EOF नहीं)।

क्लाइंट एक न्यूनतम libp2p स्वॉर्म उठाता है (request-response +
identify), 30 s कॉल टाइमआउट (20 s round trip)। डिफ़ॉल्ट `--peer`:
`/ip4/127.0.0.1/udp/4001/quic-v1/p2p/12D3KooW…` (स्थानीय dev नोड;
वास्तविक peer id सहित पूर्ण multiaddr रिले
stdout पर `p2p:` पंक्ति से घोषित)।

आंतरिक `RelayCommand` चैनल एम्बेडेड DNS
सतह के लिए बचा (`ResolveLocal`, एक in-process round trip, कभी
DHT waiter नहीं — `dns.md` देखें)।

## CLI

```
scone relay [--network testnet|mainnet|devnet] [--data-dir DIR] [--listen MULTIADDR]
            [--bootstrap MULTIADDR]... [--upnp]
            [--anchor-key PATH] [--no-anchor-auto] [--anchor-passphrase-env VAR] [--genesis FILE]
            [--topology-profile tiny|small|default|high|very-high|devtest]
            [--dns ADDR] [--dns-upstream ADDR]...
            [--metrics-addr ADDR:PORT] [--metrics-token TOKEN]
scone status [--peer MULTIADDR]
scone domains [--identity ID | --pubkey HEX] [--peer MULTIADDR]
scone domain register NAME [--identity ID] [--dir DIR] [--passphrase-env VAR] [--peer MULTIADDR]
scone domain set NAME TYPE VALUE [TTL] [--identity ID] [--peer MULTIADDR]   # express single record
scone domain update NAME --file FILE [--identity ID] [--peer MULTIADDR]
scone domain get NAME [--peer MULTIADDR]
scone domain renew NAME [--identity ID] [--peer MULTIADDR]
scone domain resolve NAME [--peer MULTIADDR]

# --peer defaults to the local relay (SCONE_PEER env overrides);
# --identity defaults to the single keystore identity when there
# is exactly one. Removed (stubs suggest replacements): show,
# submit, lookup, owned.
```

प्रत्येक नियंत्रण कमांड रिले के multiaddr (`--peer`, डिफ़ॉल्ट
स्थानीय नोड — RPC→p2p एकीकरण, अब कोई JSON TCP चैनल नहीं) से
सीधे QUIC बोलती है। रिले अनुपस्थित → स्पष्ट संदेश "p2p node
unreachable — is a relay running?", ग़ैर-शून्य exit। `--rpc`
फ़्लैग बचा है पर **अप्रचलित और उपेक्षित** (RPC सर्वर हटा;
केवल स्क्रिप्ट संगति)।

### वातावरण विन्यास (`.env`)

स्टार्टअप पर, `scone` (dotenvy से, clap पार्सिंग से पहले) मौजूदा
निर्देशिका से `.env` फ़ाइल लोड करता है: फ़ाइल गायब = मौन
no-op, कभी त्रुटि नहीं। लोड किए मान उन विकल्पों का
**वातावरणीय आधार** बनते हैं जो वेरिएबल घोषित करते हैं
(cli.rs में `env = "…"`)। **स्पष्ट CLI फ़्लैग सदा** `.env` पर
जीतता है, और `.env` अंतर्निहित डिफ़ॉल्ट पर:

```text
explicit CLI flag  >  .env variable  >  built-in default
```

रिपॉज़िटरी रूट की `.env.example` फ़ाइल प्रत्येक
वेरिएबल प्रलेखित करती है (`.env` में कॉपी कर समायोजित करें; `.env`
**gitignored** है — वास्तविक `.env` कभी commit न करें)। वेरिएबल:

| वेरिएबल | दर्पण फ़्लैग | Semantics |
|---|---|---|
| `SCONE_NETWORK` | `--network` | `testnet` (डिफ़ॉल्ट), `mainnet`, `devnet` |
| `SCONE_DATA_DIR` | `--data-dir` | data निर्देशिका (डिफ़ॉल्ट `~/.scone/<network>/`) |
| `SCONE_LISTEN` | `--listen` | P2P सुनने का multiaddr (डिफ़ॉल्ट: सभी इंटरफ़ेस, अल्पकालिक पोर्ट) |
| `SCONE_BOOTSTRAP` | `--bootstrap` | bootstrap seeds, **दोहराने योग्य** (प्रति मान एक पंक्ति — dotenvy जमा करता है); builtin seeds बदलता है |
| `SCONE_GENESIS` | `--genesis` | जेनेसिस फ़ाइल (संस्थापक नोड) |
| `SCONE_MDNS` | `--mdns` | (पुराना — mDNS खोज हटा दी गई; स्वीकृत, उपेक्षित) |
| `SCONE_UPNP` | `--upnp` | UPnP पोर्ट मैपिंग (`true` = सक्षम) |
| `SCONE_DNS_ADDR` | `--dns` | UDP DNS सर्वर bind पता |
| `SCONE_DNS_UPSTREAM` | `--dns-upstream` | रिकर्सिव फ़ॉलबैक, **दोहराने योग्य** |
| `SCONE_DNS_CACHE_LIMIT` | `--dns-cache-limit` | DNS कैश प्रविष्टि सीमा |
| `SCONE_DNS_CACHE_MAX_MEM` | `--dns-cache-max-mem` | DNS कैश मेमोरी सीमा (उदा. `64MB`) |
| `SCONE_DNS_MAX_INFLIGHT` | `--dns-max-inflight` | समवर्ती DNS प्रश्न सीमा |
| `SCONE_METRICS_ADDR` | `--metrics-addr` | metrics endpoint address (`ADDR:PORT`)|
| `SCONE_METRICS_TOKEN` | `--metrics-token` | metrics endpoint bearer token |
| `SCONE_TOPOLOGY_PROFILE` | `--topology-profile` | टोपोलॉजी profile (`tiny`/`small`/`default`/`high`/`very-high`/`devtest`; डिफ़ॉल्ट `default`) |
| `SCONE_ANCHOR_KEY` | `--anchor-key` | एंकर कीफ़ाइल (चेकपॉइंट + उत्पादन) |

`SCONE_ANCHOR_PASS` किसी विकल्प का वेरिएबल **नहीं** है: यह
**एंकर पासफ़्रेज़ धारण करने वाले वेरिएबल** का डिफ़ॉल्ट
नाम है (`--anchor-passphrase-env`, CLI स्तर पर डिफ़ॉल्ट
`SCONE_ANCHOR_PASS`) — रिले पासफ़्रेज़ उस
वेरिएबल से पढ़ता है जिसे यह फ़्लैग नाम देता है।

**एंकर auto (डिफ़ॉल्ट चालू)**: `--anchor-key` के बिना, एंकर
कीफ़ाइल data dir में स्वचालित रूप से उत्पन्न/स्थिर
(`anchor.sconekey`, पासफ़्रेज़ `SCONE_ANCHOR_PASS` यदि परिभाषित,
अन्यथा यादृच्छिक उत्पन्न और `anchor.pass` में लिखा) — रिले
उत्पादक बन जाता है जैसे ही उसकी कुंजी जीवित
डोमेन/TLD रखती है (PoS पात्रता कभी मुफ़्त नहीं)। `--no-anchor-auto`
रिले को सख़्ती से निष्क्रिय रखता है। `--anchor-passphrase-env`
का डिफ़ॉल्ट env var `SCONE_ANCHOR_PASS` है।

`--network`: डिफ़ॉल्ट BUILD का अनुसरण करता है
(mainnet-सक्षम बाइनरीज़ पर mainnet, अन्यथा testnet); `devnet`
रीसेट करने योग्य स्थानीय नेटवर्क है (अपना
`scone-devnet` जेनेसिस, लगभग-शून्य 0..=2 बिट PoW —
`~/.scone/devnet/` मिटाकर पुनः चलाना चेन रीसेट करता है)। यह
जेनेसिस, PoW कठिनाइयाँ और डिफ़ॉल्ट
data-dir (`~/.scone/<network>/`) सेट करता है; दूसरे नेटवर्क का tx
टाइप्य रूप से इनकार (`WrongNetwork`)।

ऑफ़लाइन हेक्स सतह (`tx` और `submit` subcommands) गायब
(remove-tx-surface-v2)। CLI उपयोगकर्ता पथ
(`domain register`, `domain renew`, `domains`) और M8b
परिवार को प्रथम-श्रेणी कमांड के रूप में कवर करता है: `tld open`,
`tld transfer`, `tld revoke`, `domain assign`, `domain transfer`।
`register` दावे अपनी PoW नेटवर्क कठिनाई पर माइन करते हैं
(devnet पर मिलीसेकंड)।

**कोई geo या health विकल्प मौजूद नहीं**: geozone
**बिल्ड-लागू** है (एम्बेडेड DB-IP Lite तालिका — पूरी semantics
[`dns.md`](dns.md#geodns-geographic-routing) में प्रलेखित);
भौगोलिक रूटिंग बिना किसी विकल्प के **हमेशा चालू** है, और
रिज़ॉल्वर कोई health जाँच नहीं करता — यह हस्ताक्षरित रिकॉर्ड
समुच्चय जों का त्यों सेव करता है (endpoint उपलब्धता endpoint
ऑपरेटरों का विषय है)। कोई भी
`--geo-*`/`--health-*` flag कठोर clap अस्वीकृति है।

### `domain register|update` (M5) — एक कमांड में हस्ताक्षरित लेनदेन

ये कमांड एक ही आह्वान में build → sign (keystore) → submit →
(पुष्टि) जोड़ती हैं।

- **register**: हस्ताक्षरित `RegisterDomain` बनाता है (timestamp =
  now, devnet में खाली proof), जमा करता है, ऑन-चेन
  पुष्टि की प्रतीक्षा (sequence ≥ 0, 30 s timeout)। पहले से
  पंजीकृत नाम क्लाइंट-पक्ष और रिले-पक्ष दोनों से इनकार।
- **update**: रिकॉर्ड फ़ाइल एकमात्र सत्य स्रोत है। अनुक्रम
  (ऑन-चेन + 1) और `record_hash` (फ़ाइल के रिकॉर्डों का कैननिकल
  BLAKE3) **व्युत्पन्न** होते हैं, कभी दर्ज नहीं: हस्ताक्षरकर्ता और
  सत्यापक भिन्न नहीं हो सकते। UpdateDomain पुष्टि के बाद,
  `SignedDnsRecord` (फ़ाइल के रिकॉर्ड + owner + कैननिकल
  एनकोडिंग पर Ed25519 हस्ताक्षर) `put_record` द्वारा DHT में
  प्रकाशित (जो चेन से पुनः सत्यापित करता है)। अपंजीकृत डोमेन,
  ग़ैर-स्वामी पहचान, या फ़ाइल त्रुटि साफ़ विफलता,
  चेन को छुए बिना।

#### रिकॉर्ड फ़ाइल

पाठ, प्रति पंक्ति एक रिकॉर्ड, `#` = टिप्पणी, खाली पंक्तियाँ उपेक्षित;
क्रमबद्ध कैननिकल एनकोडिंग से फ़ाइल के क्रम का कमिटेड
हैश पर कोई प्रभाव नहीं। फ़ाइल पूर्ण समुच्चय है (अपडेट
बदलता है, मिलाता नहीं)। प्रकार:

```text
A 192.0.2.1
AAAA 2001:db8::1
CNAME www.example.uip
NS ns1.example.uip
MX 10 mail.example.uip
TXT free text (spaces normalized)
```

टाइप्ड त्रुटियाँ: अज्ञात प्रकार, अमान्य IP/नाम (पंक्ति संख्या सहित),
खाली समुच्चय, प्रतिकृति, > `MAX_RECORDS_PER_SET`।

## `--genesis` (संस्थापक)

`scone relay --genesis genesis.bin` — **केवल संस्थापक नोड**
(किकस्टार्टर अभियान; फ़ाइल `scone-genesis` बाइनरी द्वारा
निर्मित)। नॉर्मेटिव नियम:

1. फ़ाइल पढ़ने से **पहले** इनकार यदि वह 1 MiB से बड़ी हो,
   फिर सख्ती से डिकोड (`decode_complete`: कोई अनुगामी बाइट
   नहीं)। ब्लॉक 0 का हैश **पुनर्गणित**, फ़ाइल से कभी नहीं पढ़ा।
2. शब्दार्थ सत्यापन: ऊँचाई 0, `consensus` माँगे नेटवर्क के
   network id (`--network`) से शुरू, कोई लेनदेन नहीं,
   कैननिकल आवंटन मैनिफ़ेस्ट। कोई भी उल्लंघन = startup पर टाइप्ड
   त्रुटि (कभी panic नहीं, डेमन शुरू नहीं होता)।
3. **कुँवारी** data-dir: चेन आवंटित जेनेसिस पर जन्म लेती है —
   आवंटन किसी भी डिस्क लेखन से **पहले** RAM में लागू, फिर
   स्थिर (seeded अवस्थाएँ + meta कुंजी तले जेनेसिस ब्लॉक)। पहला
   उत्पादित ब्लॉक इसलिए केवल आवंटित
   जेनेसिस का विस्तार कर सकता है।
4. **पहले से उत्पादक** data-dir: store का जन्म जेनेसिस
   ब्लॉक 1 का `prev_hash` है (स्वयं-प्रमाणित)। बराबर फ़ाइल हैश =
   मौन OK (विकल्प **पहले startup के बाद अनावश्यक**);
   भिन्न = टाइप्ड इनकार — नोड नेटवर्क से sync से जुड़ता है,
   `--genesis` से कभी नहीं।
5. विकल्प के **बिना** पुनर्चालन: संस्थापक-जन्म data-dir meta
   कुंजी से अपना आवंटित जेनेसिस पुनर्प्राप्त (प्रत्येक
   बूट पर RAM पुनः बाँध)।

स्थिर जेनेसिस हैश पिन (एंटी नेटवर्क-विचलन) अलग
कार्यधारा है।

### `scone-genesis` बाइनरी मैनिफ़ेस्ट (किकस्टार्टर)

`scone-genesis` बाइनरी (crate `scone-genesis`)
`genesis-manifest.json` मैनिफ़ेस्ट (`ALLOC-V1` प्रारूप) तैयार कर
`genesis.bin` उत्पन्न करती है। नॉर्मेटिव नियम:

- `scone-genesis new --network <network>` खाली
  मैनिफ़ेस्ट बनाता है; `add`/`remove`/`list` आवंटन प्रबंधित करते,
  `build` `genesis.bin` (+ हैश) देता है, `verify` पुनःसत्यापित करता
  है।
- `add domain <name>` माँगता है कि डोमेन का **TLD मैनिफ़ेस्ट में
  पहले से आवंटित हो** (अन्यथा तुरंत इनकार — चेन इसे जेनेसिस
  अनुप्रयोग पर इनकार करती); उसका
  `valid_until` **स्वचालित** है: `born_at + 1 year`
  (मैनिफ़ेस्ट की default_validity), कभी हाथ से दर्ज नहीं।
- `add tld <name>`: **डिफ़ॉल्ट रूप से खुला** (किसी का नहीं, स्व-सेवा,
  1-वर्ष फ़्लोर, वायर पर owner = शून्य — kind `0x03`);
  `--private` = निजी (owner, जेनेसिस अवधि `born_at +
  TLD_TERM_SECS` — 3 वर्ष, मानक जीवनचक्र: `RenewTld`,
  60-दिन अवधि)।
- नाम से कैननिकल क्रम (समान सामग्री → वही ब्लॉक → वही हैश,
  निवेश क्रम चाहे जो हो); प्रतिकृतियाँ इनकार।

## सीमाएँ और मेमोरी गारंटियाँ

| संसाधन | सीमा |
|---|---|
| P2P फ़्रेम | पार्सिंग से पहले `MAX_MESSAGE_LEN` (1 MiB) |
| मेमपूल | 4096 tx (आगे टाइप्ड इनकार); प्रति डोमेन `MAX_PENDING_PER_DOMAIN = 4` लंबित txs |
| मेमपूल अनुग्रह | `MAX_GRACE_ATTEMPTS = 10` ग़ैर-निश्चित कारण से बहे tx के पुनःनिवेश (40 s/चक्र पर ≈ 7 मिनट), फिर निष्कासन |
| फ़ॉर्क पार्किंग | `PARKED_CAP = 2048` प्रतिस्पर्धी ब्लॉक |
| संचित चेकपॉइंट | `MAX_PENDING_CHECKPOINTS = 256` (निष्कासन: सबसे छोटा डेटा हैश) |
| समवर्ती DHT waiters | `MAX_DHT_WAITERS` (256); DHT खोज 30 s तक सीमित (`DHT_LOOKUP_TIMEOUT`) |
| प्रति sync उत्तर ब्लॉक | प्रति round-trip 1; प्रति पीयर ≤ 10,000 बैच (`MAX_SYNC_ROUNDS`); अनुरोध पर प्रोटोकॉल सीमा `MAX_BLOCKS_PER_REQUEST = 128` |
| DHT रिकॉर्ड | `MAX_DHT_CACHE_ENTRY` (64 KiB) |
| स्थायी DHT कैश (डिस्क) | दोहरी LRU सीमा: `DHT_CACHE_MAX_ENTRIES` (100,000 प्रविष्टियाँ) और `DHT_CACHE_MAX_BYTES` (2 GiB) — जो पहले छुए वह कुंजी क्रम से निकालता; GC निष्कासन ब्लॉक के उसी लेनदेन में (चेन GC द्वारा मारा डोमेन कभी अनाथ प्रविष्टि नहीं छोड़ता) |
| DHT निगेटिव कैश | `DHT_NEGATIVE_TTL_SECS` (30 s), FIFO cap `DHT_NEGATIVE_CAP` (4,096) |
| RAM में चेन | जेनेसिस + tip खिड़की (इतिहास store से सेव) |

एंटी-लूप नियम (H2): लेनदेन केवल तभी पुनःप्रसारित जब वह
मेमपूल में **वास्तव में डाला गया** हो (पहली दृष्टि); प्रतिकृति
(पूल में या चेन में पहले से) कभी पुनःरिले नहीं। पहले से
कैननिकल ब्लॉक `push_block` से पहले पहचाना और उपेक्षित
(पुनः-store नहीं, पुनःप्रसारण नहीं)।

खुले निर्णय: रिले की स्थायी नेटवर्क पहचान, प्रसारण
gossip (वर्तमान: प्रत्येक ज्ञात पीयर को unicast request-response)।
प्रारंभिक लेखन से हल: वास्तविक फ़ॉर्क चयन (पार्किंग +
reconsider_parked, ऊपर), PoW सहमति (हस्ताक्षरित उत्पादक +
एंकर समिति), DNS रिज़ॉल्वर (`--dns` सतह, dns.md
देखें), नियंत्रण चैनल (RPC→p2p एकीकरण: अब कोई JSON
TCP नहीं)।

## सुविधाजनक बंद (SIGTERM / SIGINT)

`scone relay` डेमन (foreground) `tokio::signal` से SIGTERM और
SIGINT पकड़ता है: संकेत पर, `Relay::run` future रद्द
(बाइनरी में `tokio::select!`), जो libp2p Swarm छोड़ता और QUIC
sockets बंद करता है। फिर tokio runtime सीमित बंद (≤ 2 s) से
रुकता है जो अलग किए
DNS tasks (UDP/TCP) छोड़ता और उनकी sockets बंद करता है — लक्ष्य
कि `kill` कोई अनाथ UDP socket न छोड़े (testnet 60 पर देखा
मामला)। संकेत-चालित बंद `relay shutting down (signal)` लॉग
(INFO, stderr) और exit code **0** से निकलता है: माँगा बंद
विफलता नहीं। Handler स्थापना विफल हो तो संकेत का डिफ़ॉल्ट
व्यवहार लागू (अवक्रमण, कभी panic नहीं)।


## Tor परिवहन (खुला मुद्दा)

QUIC/UDP सादे SOCKS प्रॉक्सी पार नहीं करता, इसलिए रिले आज
Tor पर सरलता से नहीं चल सकते। नेटिव libp2p Tor परिवहन
(onion-पते वाला रिले) अगला ट्रैक किया पथ है — वर्तमान
स्थिति और `.tor` CNAME पुल (छिपी सेवाओं के लिए पठनीय नाम,
आज समर्थित) के लिए वेबसाइट की सेंसरशिप-प्रतिरोध गाइड देखें।

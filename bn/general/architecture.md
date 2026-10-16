# স্থাপত্য (আর্কিটেকচার)

## প্রথমে সহজ ভাষায়

বাকি সব বুঝতে পাঁচটি ধারণা যথেষ্ট:

1. **একটি নাম** — ইন্টারনেটে একটি ঠিকানা, যেমন `mysite.scone`।
2. **আপনার ২৪-শব্দের বাক্যাংশ** — আপনার পরিচয়। এটিই সেই দলিল যা
   প্রমাণ করে একটি নাম আপনার, এবং আপনার করা প্রতিটি পরিবর্তনে স্বাক্ষর করে।
3. **পাবলিক রেকর্ড বই** (ব্লকচেইন) — কে কী মালিক তার ভাগ করা,
   পাবলিক খাতা। কেউ এটি নিয়ন্ত্রণ করে না, কেউ
   এটি পুনর্লিখন করতে পারে না।
4. **মাইনিং** — কীভাবে আপনি একটি নাম সংরক্ষণ করবেন: আপনার কম্পিউটার কিছু
   কাজ করে, সবার জন্য ন্যায্য ও সমান। কোনো লাইন নেই, কোনো মধ্যস্থতাকারী নেই।
5. **একটি রিলে** — এমন একটি কম্পিউটার যা বইয়ের একটি কপি রেখে এবং প্রশ্নের
   উত্তর দিয়ে নেটওয়ার্ক চালায়। যে কেউ একটি
   চালাতে পারে; যত বেশি থাকবে, নেটওয়ার্ক তত শক্তিশালী হবে।

এই পৃষ্ঠার বাকি অংশ গভীরে যায় — আপনি এখানেই থামতে পারেন এবং
ইতিমধ্যে প্রকল্পটি বুঝে গেছেন।

## পরিচালনা নীতি

Scone কর্তৃত্ব ও প্রাপ্যতাকে পৃথক করে:

- **ব্লকচেইন** হলো মালিকানা এবং পরিবর্তনের ক্রমের সত্যের উৎস;
- **DHT** হলো একটি বিতরণ/রেপ্লিকেশন ব্যবস্থা, এটি কোনো কর্তৃত্ব **নয়**;
- **লোকাল স্টোরেজ** নোডের অবস্থা, ইনডেক্স এবং ক্যাশ ধারণ করে।

একটি নোডকে অবশ্যই DHT থেকে আসা যেকোনো ডেটাকে সম্ভাব্য
ক্ষতিকর হিসেবে বিবেচনা করতে হবে: এটি শুধুমাত্র ক্রিপ্টোগ্রাফিক
যাচাইয়ের পরে গৃহীত হয় (owner সিগনেচার, চেইনের ownership /
sequence / record_hash-এর সাথে মিল)।

## দায়িত্বের বিভাজন

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

## ক্রেট ও ডিপেন্ডেন্সি

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

### সিদ্ধান্ত: `scone-core`, `scone-crypto`-র উপর নির্ভর করে

`DomainId` অবশ্যই ভিত্তি থেকে গণনাযোগ্য হতে হবে
(`BLAKE3-256(prefix || canonical name)`), কিন্তু কংক্রিট ক্রিপ্টোগ্রাফি
কোরকে দূষিত করবে না। সমাধান:

- `scone-crypto` হলো সর্বনিম্ন স্তর: বাইটের উপর বিশুদ্ধভাবে ডিটারমিনিস্টিক
  ফাংশন (প্লাস কী জেনারেশনের জন্য OS RNG), শূন্য কোর টাইপ,
  কোনো নেটওয়ার্ক/ফাইলসিস্টেম/অ্যাসিঙ্ক ডিপেন্ডেন্সি নেই → কোনো বৃত্তীয়তা সম্ভব নয়;
- `scone-core` এটি শুধুমাত্র হ্যাশিংয়ের জন্য ব্যবহার করে;
- Ed25519 সিগনেচার ও কী `scone-crypto`-তে থাকে এবং
  বাইটের উপর কাজ করে (ক্যানোনিকাল পেলোড `scone-protocol` উৎপাদন করে)।

### প্রতিটি ক্রেটের ভূমিকা

**scone-core** — নাম (`TldName`, `DomainName`), শনাক্তকারী
(`DomainId`, `OwnerId`, `PublicKeyRef`), DNS ডেটা (`DnsRecord`,
`SignedDnsRecord`), লেনদেন (`RegisterDomain`, `UpdateDomain`)।
ডিটারমিনিস্টিক ও পোর্টেবল। এই ক্রেটে নিষিদ্ধ: libp2p, redb,
tokio/async, নেটওয়ার্ক, ফাইল, OS, DNS সার্ভার, ব্লকচেইন লজিক।

**scone-crypto** — ক্রিপ্টোগ্রাফিক প্রিমিটিভ। আজ: `hash256`
(BLAKE3), Ed25519 সিগনেচার ও কী (`SigningKey`,
`PublicKey`, `Signature`, কঠোর ব্যাচ ভেরিফিকেশন), রেকর্ড
গ্রুপের AEAD এনভেলপ (XChaCha20-Poly1305)। পরিকল্পিত: মার্কেল ট্রি।
নিয়ম: কোনো স্বশিক্ষিত অ্যালগরিদম নয়, শুধুমাত্র অডিটেড ডিপেন্ডেন্সি।

**scone-protocol** — প্রোটোকলের বাইনারি ফরম্যাট: লেনদেন, DNS
রেকর্ড, ব্লক (`Block`/`BlockHeader`, একটি অপাস কনসেনসাস ফিল্ড সহ)
এবং P2P বার্তার ক্যানোনিকাল এনকোডিং/ডিকোডিং (ন্যূনতম LEB128
ভ্যারিন্ট, খালি ৩২-বাইট শনাক্তকারী), অ্যালোকেশনের আগে
সাইজ লিমিট যাচাই, অ-ক্যানোনিকাল রূপ প্রত্যাখ্যান, এবং ভার্সনিং
(`PROTOCOL_VERSION`) সহ। কোনো কনসেনসাস, নেটওয়ার্ক বা স্টোরেজ লজিক নেই।
ফরম্যাট রেফারেন্স: `/docs/en/technical/protocol.md`।

**scone-blockchain** — ব্লকচেইন লজিক ও ক্যানোনিকাল অবস্থা:
লেনদেন শনাক্তকরণ (`TxId`), ক্রমবদ্ধ মার্কেল ট্রি,
`BlockHash`, ডিটারমিনিস্টিক জেনেসিস, ব্লক ও লেনদেন ভ্যালিডেশন,
অবস্থা প্রয়োগের নিয়ম (`DomainState`/`ChainState`, মেমরিতে,
`DomainId` দিয়ে সরাসরি অ্যাক্সেস), ফর্ক সনাক্তকরণ ও ব্রাঞ্চ নির্বাচন
(`fork_choice`: ফাইনালিটি ফ্লোর + দীর্ঘতম চেইন + সর্বনিম্ন-হ্যাশ
টাই-ব্রেক), চেকপয়েন্ট ফাইনালিটি, এবং কনসেনসাস অ্যাবস্ট্রাকশন
(`trait Consensus`, ডিফল্টে পারমিসিভ ইমপ্লিমেন্টেশন)। রেজিস্ট্রেশন
PoW (মাইন/ভেরিফাই, প্রতি-নেটওয়ার্ক ডিফিকাল্টি) `scone-core`-এ থাকে।
এই ক্রেটে নিষিদ্ধ: নেটওয়ার্ক, DHT, স্টোরেজ (redb),
DNS সার্ভার। রেফারেন্স: `/docs/en/technical/blockchain.md`।

**scone-storage** — লোকাল পারসিস্টেন্স (শিপড)। কংক্রিট ব্যাকএন্ড
হলো `redb` (`NodeStore`), একটি ন্যূনতম trait-এর আড়ালে লুকানো যাতে
ব্লকচেইন/DHT/DNS কখনো সরাসরি redb-এর সাথে যুক্ত না হয়; আরেকটি
ব্যাকএন্ড (ডিস্ট্রিবিউটেড KV) প্লাগ ইন করা যেত। এর মধ্যে রয়েছে
পেজিনেটেড KV স্টেট ব্যাকএন্ড, ব্লক লগ, ভেরিফায়েড স্ন্যাপশট (হ্যাশড
ম্যানিফেস্ট + পেজ) এবং রিস্টার্ট রিস্টোর/রিপ্লে পাথ। রেফারেন্স:
`/docs/en/technical/storage.md`।

**scone-network** — নোডের p2p রিলে: libp2p/QUIC স্ওয়ার্ম,
লেনদেন ও ব্লক রিলে (সিস্টেম্যাটিক লোকাল ভ্যালিডেশন), বাউন্ডেড
মেমপুল, বাউন্ডেড ব্লক সিঙ্ক (১২৮ ব্লক/রিকোয়েস্ট), `SignedDnsRecord`-এর
জন্য Kademlia DHT, UDP+TCP DNS সার্ভার (GeoDNS সর্বদা চালু —
এমবেডেড GeoIP টেবিল, রিকার্সিভ ফলব্যাক), অ্যাঙ্করদের
চেকপয়েন্ট/হার্টবিট লুপ (CLI রিলের সাথে সরাসরি p2p QUIC-এ কথা বলে,
আর কোনো লোকাল RPC নেই)। রিচেবিলিটি ও ডিসকভারি: AutoNAT
v2 (reachable/NATed রায়), NATed নোডের জন্য circuit-relay v2 + DCUtR,
mDNS (শূন্য-কনফিগ LAN ডিসকভারি, devnet-এ ডিফল্টে ON)
এবং UPnP (রাউটার দ্বারা QUIC পোর্ট ম্যাপিং, অপ্ট-ইন)। একটি `.env` ফাইলের মাধ্যমে কনফিগারেশন (দেখুন `.env.example`)।
রেফারেন্স: `/docs/en/technical/relay.md`।

**scone-keystore** — Ed25519 সাইনিং কী-এর এনক্রিপ্টেড স্টোরেজ
(`.sconekey` ফাইল: Argon2id + XChaCha20-Poly1305)। এই ক্রেট
ফাইলসিস্টেমের মালিক যাতে `scone-crypto` বিশুদ্ধ থাকতে পারে। রেফারেন্স:
`/docs/en/technical/keystore.md`।

**scone-topology** — লোকাল p2p টোপোলজি অপ্টিমাইজেশনের বিশুদ্ধ (PURE)
ইঞ্জিন: শূন্য নেটওয়ার্ক, শূন্য ফাইলসিস্টেম, শূন্য অ্যাসিঙ্ক। এটি libp2p
বা স্ওয়ার্ম কিছুই জানে না: `scone-network` এটিকে পর্যবেক্ষণ দেয়
(ping RTT, সংযোগ) এবং প্রতিক্রিয়ায় পাড়া-পরিচর্যার সিদ্ধান্ত পায়।

**scone-genesis** — মেইননেট জেনেসিস প্রস্তুতির ওয়ান-শট টুল
বাইনারি (কিকস্টার্টার): ভার্সনড JSON ম্যানিফেস্ট, অ্যালোকেশন,
ডিটারমিনিস্টিক ব্লক।

**scone** — বাইনারি/CLI। একটি ক্রেটের উপর তখনই নির্ভর করে যখন
প্রকৃতপক্ষে ব্যবহার করে (আজ: রিলের জন্য scone-network,
scone-blockchain, scone-storage, scone-keystore, scone-protocol,
scone-core, scone-crypto, scone-topology)।

## নোড

একটি Scone নোড একত্রিত করে:

1. একটি ব্লকচেইন (কনসেনসাস ভ্যালিডেশন, মালিকানার অবস্থা);
2. DHT রাউটিং (`SignedDnsRecord`-এর পাবলিকেশন/রিট্রিভাল);
3. লোকাল স্টোরেজ (অবস্থা, ইনডেক্স, ক্যাশ);
4. সামনে একটি রিজলভার / DNS সার্ভার।

### নতুন নোড শুরু করা (স্টেট বুটস্ট্র্যাপ, P0.3/P0.4)

একটি নতুন নোড জেনেসিস রিপ্লে করে না: এটি একটি ভেরিফায়েড
ফাইনালাইজড অবস্থা থেকে শুরু হয়।

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

দুটি নোড প্রোফাইল:

- **আর্কাইভ নোড** (অপারেটর ফ্ল্যাগ): সব ঐতিহাসিক ব্লক রাখে,
  নেটওয়ার্ককে সেবা দেয়, বিদ্যমান পাথ দিয়ে শুরু হয়;
- **ফুল-স্টেট নোড**: H-তে সম্পূর্ণ অবস্থা ও সাফিক্স ধারণ করে,
  কিন্তু H-এর নিচের ইতিহাস নয় — এটি ভবিষ্যৎ ব্লক ভ্যালিডেট করে ও
  অবস্থার সেবা দেয়, ইতিহাসের সেবা দেয় না।

এই পাথটি অপ্ট-ইন: একটি ভেরিফায়েড স্ন্যাপশট ইমপোর্ট করা একটি স্পষ্ট
অপারেটর ক্রিয়া (`import_verified_snapshot`), কখনো স্বয়ংক্রিয় নয় —
একটি বিদ্যমান নোডের ডিফল্ট আচরণ পরিবর্তন হয় না।
নরমেটিভ বিস্তারিত: `/docs/en/technical/storage.md`, সেকশন
"VERIFIED state snapshot" এবং "State bootstrap"।

## DNS রেজোলিউশন প্রবাহ

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

DNS স্ন্যাপশট (প্রতিটি লাইভ ডোমেইনের অস্তিত্ব + মেয়াদ,
প্রতিটি প্রয়োগকৃত ব্লকের পরে প্রকাশিত) নিশ্চয়তা দেয় যে কোনো
অস্তিত্বের সিদ্ধান্ত কখনো রিলের রান লুপের মধ্য দিয়ে যায় না:
কোর শুধুমাত্র প্রকৃত রেকর্ড আনার সময় স্পর্শ করা হয়।

## বিবর্ধনযোগ্যতা

- ভার্সনড প্রিইমেজ (`SCONE-DOMAIN-V1`, `SCONE-PUBKEY-V1`,
  `SCONE-OWNER-V1`, …): একটি ডেরিভেশন নিয়ম পরিবর্তন করলে বিদ্যমানটি
  ভাঙার বদলে একটি নতুন নেমস্পেস তৈরি হয়;
- `RecordData` ভ্যারিয়েন্টের মাধ্যমে সম্প্রসারণযোগ্য + পড়ার সময়
  অজানা টাইপের জন্য `Unknown { type_code, data }`;
- ওয়্যার ফরম্যাট বিরতির জন্য `PROTOCOL_VERSION` (`scone-protocol`);
- চেইন শুধুমাত্র হ্যাশ সংরক্ষণ করে: বড় ডেটা DHT-তে থাকে।

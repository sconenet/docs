# Architecture

## Guiding principle

Scone separates authority from availability:

- the **blockchain** is the source of truth for ownership and the order of changes;
- the **DHT** is a distribution/replication system, it is **not** an authority;
- **local storage** holds the node's state, indexes, and caches.

A node must treat any data coming from the DHT as potentially
malicious: it is accepted only after cryptographic verification
(owner signature, match against the ownership / sequence /
record_hash of the chain).

## Separation of responsibilities

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

## Crates and dependencies

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

### Decision: `scone-core` depends on `scone-crypto`

The `DomainId` must be computable from the foundation up
(`BLAKE3-256(prefix || canonical name)`), but concrete cryptography
must not contaminate the core. Resolution:

- `scone-crypto` is the lowest layer: purely deterministic functions
  over bytes (plus the OS RNG for key generation), zero core types,
  no network/filesystem/async dependencies → no circularity possible;
- `scone-core` uses it only for hashing;
- Ed25519 signatures and keys live in `scone-crypto` and operate on
  bytes (canonical payloads produced by `scone-protocol`).

### Role of each crate

**scone-core** — names (`TldName`, `DomainName`), identifiers
(`DomainId`, `OwnerId`, `PublicKeyRef`), DNS data (`DnsRecord`,
`SignedDnsRecord`), transactions (`RegisterDomain`, `UpdateDomain`).
Deterministic and portable. Forbidden in this crate: libp2p, redb,
tokio/async, network, files, OS, DNS server, blockchain logic.

**scone-crypto** — cryptographic primitives. Today: `hash256`
(BLAKE3), Ed25519 signatures and keys (`SigningKey`,
`PublicKey`, `Signature`, strict batch verification), AEAD envelope
of record groups (XChaCha20-Poly1305). Planned: Merkle tree.
Rule: no homegrown algorithms, audited dependencies only.

**scone-protocol** — binary format of the protocol: canonical
encoding/decoding (minimal LEB128 varint, bare 32-byte identifiers)
of transactions, DNS records, blocks (`Block`/`BlockHeader` with an
opaque consensus field) and P2P messages, with size limits checked
before allocation, rejection of non-canonical forms, and versioning
(`PROTOCOL_VERSION`). No consensus, network, or storage logic.
Format reference: `/docs/en/technical/protocol.md`.

**scone-blockchain** — blockchain logic and canonical state:
transaction identification (`TxId`), ordered Merkle tree,
`BlockHash`, deterministic genesis, block and transaction validation,
state application rules (`DomainState`/`ChainState`, in memory,
direct access by `DomainId`), fork detection and branch choice
(`fork_choice`: finality floor + longest chain + lowest-hash
tie-break), checkpoint finality, and consensus abstraction
(`trait Consensus`, permissive implementation by default). The
registration PoW (mine/verify, per-network difficulties) lives in
`scone-core`. Forbidden in this crate: network, DHT, storage (redb),
DNS server. Reference: `/docs/en/technical/blockchain.md`.

**scone-storage** — local persistence (shipped). The concrete backend is
`redb` (`NodeStore`), hidden behind a minimal trait so that
blockchain/DHT/DNS are never directly coupled to redb; another
backend (distributed KV) could be plugged in. Includes the paginated
KV state backend, the block log, verified snapshots (hashed manifest
+ pages) and the restart restore/replay path. Reference:
`/docs/en/technical/storage.md`.

**scone-network** — the node's p2p relay: libp2p/QUIC swarm,
transaction and block relay (systematic local validation), bounded
mempool, bounded block sync (128 blocks/request), Kademlia DHT for
`SignedDnsRecord`, UDP DNS server (optional GeoDNS, recursive
fallback), the anchors' checkpoint/heartbeat loop (the CLI speaks
direct p2p QUIC to the relay, no local RPC anymore). Reference:
`/docs/en/technical/relay.md`.

**scone-keystore** — encrypted storage of Ed25519 signing keys
(`.sconekey` files: Argon2id + XChaCha20-Poly1305). This crate owns
the filesystem so `scone-crypto` can stay pure. Reference:
`/docs/en/technical/keystore.md`.

**scone-topology** — PURE engine for local p2p topology optimization:
zero network, zero filesystem, zero async. It knows neither libp2p
nor the swarm: `scone-network` feeds it observations (ping RTT,
connections) and receives neighborhood management decisions.

**scone-genesis** — one-shot tool binary (kickstarter) for mainnet
genesis preparation: versioned JSON manifest, allocations,
deterministic block.

**scone** — binary/CLI. Depends on a crate only when it actually
uses it (today: scone-network for the relay, scone-blockchain,
scone-storage, scone-keystore, scone-protocol, scone-core,
scone-crypto, scone-topology).

## The node

A Scone node combines:

1. a blockchain (consensus validation, ownership state);
2. DHT routing (publication/retrieval of `SignedDnsRecord`);
3. local storage (state, indexes, caches);
4. a resolver / DNS server at the front.

### Starting a new node (state bootstrap, P0.3/P0.4)

A new node does NOT replay genesis: it starts from a verified
finalized state.

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

Two node profiles:

- **archive node** (operator flag): keeps all historical blocks,
  serves them to the network, starts via the existing paths;
- **full-state node**: holds the full state at H and the suffix,
  but not the history below H — it validates future blocks and
  serves state, it does not serve history.

This path is OPT-IN: importing a verified snapshot is an explicit
operator action (`import_verified_snapshot`), never automatic —
the default behavior of an existing node does not change.
Normative details: `/docs/en/technical/storage.md`, sections
"VERIFIED state snapshot" and "State bootstrap".

## DNS resolution flow

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

The DNS snapshot (existence + expiry of every live domain,
published after each applied block) guarantees that an existence
decision never goes through the relay's run loop: the core is
only touched for actual record fetches.

## Evolvability

- versioned preimages (`SCONE-DOMAIN-V1`, `SCONE-PUBKEY-V1`,
  `SCONE-OWNER-V1`, …): changing a derivation rule creates a new
  namespace rather than breaking the existing one;
- `RecordData` extensible via variants + `Unknown { type_code, data }`
  for types not known at read time;
- `PROTOCOL_VERSION` (`scone-protocol`) for wire format breaks;
- the chain stores only hashes: large data stays in the DHT.

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
scone-crypto    ← lowest layer: pure primitives over bytes
    ▲
scone-core      ← protocol types (depends on scone-crypto)
    ▲
scone-protocol  ← wire/blockchain binary format
    ▲
scone-blockchain← blockchain logic, canonical state (in memory)
    ▲
scone-storage   ← local persistence: redb backend + verified snapshots
    ▲
scone           ← binary / application
```

### Decision: `scone-core` depends on `scone-crypto`

The `DomainId` must be computable from the foundation up
(`BLAKE3-256(prefix || canonical name)`), but concrete cryptography
must not contaminate the core. Resolution:

- `scone-crypto` is the lowest layer: purely deterministic functions
  over bytes, zero core types, no network/system dependencies →
  no circularity possible;
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
(BLAKE3). Planned: signatures (Ed25519), keys, proof of work, Merkle
tree. Rule: no homegrown algorithms, audited dependencies only.

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
direct access by `DomainId`), minimal fork detection, and consensus
abstraction (`trait Consensus`, permissive implementation by
default). Forbidden in this crate: network, DHT, storage (redb),
DNS server, concrete consensus (PoW/difficulty/fork choice).
Reference: `/docs/en/technical/blockchain.md`.

**scone-storage** — local persistence (shipped). The concrete backend is
`redb` (`NodeStore`), hidden behind a minimal trait so that
blockchain/DHT/DNS are never directly coupled to redb; another
backend (distributed KV) could be plugged in. Includes the paginated
KV state backend, the block log, verified snapshots (hashed manifest
+ pages) and the restart restore/replay path. Reference:
`/docs/en/technical/storage.md`.

**scone** — binary/CLI. Depends on a crate only when it actually
uses it (today: scone-network for the relay, scone-blockchain,
scone-storage, scone-keystore, scone-protocol, scone-core).

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
    ├── local cache (SignedDnsRecord) ──► verification (signature, owner,
    │                                      sequence, record_hash vs chain)
    ├── DHT get(DomainId) ──────────────► same verification
    ▼
DNS answer (if and only if valid)
```

## Evolvability

- versioned preimages (`SCONE-DOMAIN-V1`, `SCONE-PUBKEY-V1`,
  `SCONE-OWNER-V1`, …): changing a derivation rule creates a new
  namespace rather than breaking the existing one;
- `RecordData` extensible via variants + `Unknown { type_code, data }`
  for types not known at read time;
- `PROTOCOL_VERSION` (`scone-protocol`) for wire format breaks;
- the chain stores only hashes: large data stays in the DHT.

# Scone Documentation

> All documentation under `docs/` is **NORMATIVE**: it must enable an
> independent implementation without reading the Rust. Any doc/code
> divergence is a bug (identify the source of truth, fix it).

Two parts:

- **`general/`** — how it works: the protocol's concepts and rules,
  accessible without reading the details of the formats;
- **`technical/`** — detailed specifications for developers: exact
  binary formats, hash formulas, bounds, APIs, and the behavior of
  each component.

## `general/` — how it works

| Document | Content |
|---|---|
| [architecture.md](general/architecture.md) | Separation of authority (blockchain) / availability (DHT) / local storage; roles of the crates; DNS resolution flow; evolvability |
| [naming.md](general/naming.md) | Naming rules: LDH alphabet, TLDs, domains and subdomains, canonicalization, `DomainId` derivation and hexadecimal representation |

## `technical/` — developer specifications

| Document | Content |
|---|---|
| [protocol.md](technical/protocol.md) | Canonical wire binary format: LEB128 varints, identifiers, DNS records, blocks, P2P messages, limits, versioning |
| [transactions.md](technical/transactions.md) | Transaction formats v1: signed forms (fields, wire order — RegisterDomain carries the name, arbitrary discriminants, owner derivation, signed payload `SCONE-TX-SIG-V1`) and the four signature-less pow-to-owner forms (the mined proof IS the authorization), validation rules |
| [blockchain.md](technical/blockchain.md) | Blockchain layer: `TxId`, Merkle tree, `BlockHash`, genesis, canonical state, block validation, consensus abstraction |
| [keystore.md](technical/keystore.md) | `.sconekey` keyfile format (Argon2id + XChaCha20-Poly1305), `scone-keystore` API, `identity` commands |
| [storage.md](technical/storage.md) | Local redb persistence: table schema, atomic deltas, memory policy, trust model, DoS bounds |
| [relay.md](technical/relay.md) | Network relay (`scone-network`): libp2p protocols, peer discovery (seeds, directory, UPnP), reachability (AutoNAT, circuit-relay, DCUtR), mempool, sync, Kademlia DHT, local topology, Prometheus metrics, DNS snapshot, full CLI, `.env` configuration |
| [dns.md](technical/dns.md) | UDP + TCP DNS server (M6/M7b): authoritative DNS snapshot, 16-way sharded hot-path cache (shared entries, allocation-free lookup, packed records, name compression), RFC 1035 codec, TCP RFC 7766 (512-byte UDP / 64 KiB TCP budget, TC bit), DHT negative cache, recursive fallback, bounds |
| [cli.md](technical/cli.md) | `scone` binary: logging and verbosity (stdout/stderr, `-v` levels, `RUST_LOG`), environment configuration via `.env` (precedence flag > `.env` > default) |
| [security-model.md](technical/security-model.md) | Security model: committee selection, seed anti-grinding (analysis: VRF not necessary), equivocation and slashing |
| [audit-methodologie.md](technical/audit-methodologie.md) | Security audit methodology: divergences, source of truth, correction passes |
| [rfc-proof-of-service.md](technical/rfc-proof-of-service.md) | Proof-of-service RFC: operator heartbeats, operation delegation, committee election among fresh seats |

## Cross-reference conventions

Internal documents and Rust code comments reference these pages by
absolute path from the repository root
(e.g. `/docs/en/technical/protocol.md`).

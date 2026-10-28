# Protocol

Reference document for the Scone protocol: binary format, canonical
encoding, transactions, DNS records, blocks, P2P messages, limits and
version compatibility. The reference implementation is the
[`scone-protocol`](../../../crates/scone-protocol) crate.

Items marked **(provisional)** are not final.

## Protocol version

- constant: `PROTOCOL_VERSION: u32 = 1` (`scone-protocol`);
- carried by `Hello.version` (handshake) and `BlockHeader.version`;
- compatibility rules:
  - received version **higher** than the local version → reject
    (`ProtocolError::UnsupportedVersion`); the node does not guess the
    format of a version it does not know;
  - version `0` → invalid;
  - there is only one set of formats: the transaction version bytes
    (`0x01`) and the block version must match the local values
    **exactly**;
- no major/minor split at this stage: a single integer, incremented on
  any wire format break.

## Binary format principles

- binary, never JSON for protocol data;
- deterministic and canonical: two logically equal objects produce
  **exactly the same bytes**;
- sequential parsing, allocation-free where possible (byte borrows);
- all network input is untrusted: lengths and counters are checked
  **before** any allocation; decoding never panics.

### Endianness

- all integers are LEB128 varints (7-bit groups, least significant
  first): native endianness does not apply;
- IPv4/IPv6 addresses and 32-byte identifiers: raw byte arrays, no
  length prefix;
- there is no fixed-size wide integer in the format: a single integer
  encoding rule.

### Varint (minimal LEB128)

Each byte carries 7 bits of data; the high bit is 1 on every byte
except the last. A `u64` fits in 1 to 10 bytes.

| value | bytes (hex) |
|---|---|
| 0 | `00` |
| 1 | `01` |
| 127 | `7f` |
| 128 | `80 01` |
| 255 | `ff 01` |
| 16,384 | `80 80 01` |
| 2^32 | `80 80 80 80 10` |
| u64::MAX | `ff ff ff ff ff ff ff ff ff 01` |

The encoding must be **minimal**: oversized forms are rejected at
decoding (`InvalidVarint`). This is the necessary condition for a
value to have only one valid byte sequence (canonical encoding).
Rejections: `80 00` (non-minimal), `ff×9 02` (u64 overflow), `ff×10`
(too long).

### Strings and bytes

- `bytes`: varint length + raw bytes;
- `str`: same, with mandatory UTF-8 decoding;
- every length is capped (see [Limits](#limits)) and checked before
  reading.

### Identifiers

| Type | Wire representation |
|---|---|
| `DomainId` | 32 raw bytes |
| `TldId` | 32 raw bytes |
| `OwnerId` | 32 raw bytes |
| `PublicKeyRef` | 32 raw bytes |
| `RecordHash` | 32 raw bytes |
| `BlockHash` | 32 raw bytes |
| `MerkleRoot` | 32 raw bytes |
| `DomainName` | `str` (≤ 253 bytes, revalidated by `scone-core`) |

Derivations (`BLAKE3-256` + `SCONE-DOMAIN-V1` prefixes, etc.) remain
defined by `scone-core` (see `/docs/en/general/naming.md`).

## Transactions

```text
Transaction = disc u8 || version u8 || payload
```

| Discriminant | Type |
|---|---|
| `0x21` | `RegisterDomain` |
| `0x2A` | `RegisterTldPow` |
| `0x52` | `UpdateDomain` |
| `0x57` | `RenewTldPow` |
| `0x84` | `RegisterDomainPow` |
| `0x93` | `RegisterTld` |
| `0xC3` | `RenewDomainPow` |

Each discriminant is an arbitrary opaque constant (no logical
sequence; `0x00` never valid) — full normative table in
`/docs/en/technical/transactions.md`.

The `version` byte (current value `0x01`) follows the discriminant:
any other value is explicitly rejected
(`UnsupportedVersion(value)`). Most transactions are signed
(Ed25519) — EXCEPT the four signature-less pow-to-owner forms
(`0x2A`, `0x57`, `0x84`, `0xC3`): the mined proof IS the
authorization, and its challenge binds the beneficiary key and the
network. Full specification, signed payload and validation rules in
**`/docs/en/technical/transactions.md`** (normative).

Since the 2026-09 wire change, **transactions no longer carry a
`network` field**: the `network_id` (`str` ≤ 16) is injected into
the **signing payload** (signed forms) or the **PoW challenge**
(pow-to-owner forms) — a tx made for one network never carries
over to another (see `/docs/en/technical/blockchain.md`).

### REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

`RegisterDomain` carries the **canonical name in cleartext**, followed
by its `DomainId` — the consistency `domain_id == DomainId(name)` is
re-checked at decoding (recomputed, never trusted). `proof` carries
the registration PoW (20 bytes: nonce + epoch + difficulty — epoch
salt `epoch = height/144` of the applying block, verified STRICTLY at
application against the network constant and the block's epoch; typed
rejection `ProofEpochMismatch` on mismatch — M8b, see
`transactions.md`). `public_key` is the signer's Ed25519 key (exactly
32 bytes); `owner` must be the BLAKE3 derivation of that key
(recomputed, never trusted); `signature` is exactly 64 bytes.

### REGISTER_TLD (0x93)

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

Claim of a TLD (registry separate from the domain registry, M7): the
TLD is **carried in cleartext** like a domain's name (M8b — the PoW
challenge derives from the name), and `tld_id` is the `SCONE-TLD-V1`
derivation of that name, disjoint from every `DomainId` by
construction. State rules (M8b): TLD free + not ICANN-reserved
(`IcannTldReserved` — ICANN root TLDs are claimable by no one) + PoW
at the **length-based** difficulty of the label (`pow_length` curve —
see `blockchain.md`); claimed is closed (assign-only).

### UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

Invariant re-checked at decoding: `sequence > 0` and `owner` ==
derivation of `public_key`. No `timestamp`: ordering goes through
`sequence`. The transaction carries only compact references — the
full DNS content stays in the DHT (see below).

## DNS records (DHT side)

Full DNS records never enter the blockchain: they are exchanged via
the DHT (`GetRecord`/`Record` messages) and committed on-chain by
their `RecordHash` only.

### RecordData

```text
RecordData = type.v || ttl.v || payload(type)
```

The type codes are the **real DNS codes** (IANA). The TTL (seconds,
varint) follows the type code and covers the whole record: bounds
`MIN_TTL = 1` .. `MAX_TTL = 2,592,000` (30 days; default
`DEFAULT_TTL = 86,400`).

| Code | Type | Payload |
|---|---|---|
| 1 | A | 4 bytes |
| 2 | NS | `name` |
| 5 | CNAME | `name` |
| 12 | PTR | `name` |
| 15 | MX | `preference.v`, `name` |
| 16 | TXT | `str` ≤ 4096 |
| 28 | AAAA | 16 bytes |
| 33 | SRV | `priority.v`, `weight.v`, `port.v`, `target(name)` |
| 257 | CAA | `flags u8`, `tag` (1–15 B), `value` (`str` ≤ 4096) |
| 32768 | GEOROUTE | GeoRoute (see below) |
| other | Unknown | code.v + `bytes` ≤ 4096 |

Example: `A 192.0.2.1` (TTL 86,400) → `01 80 a3 05 c0 00 02 01`
(type, TTL varint, address).

An `Unknown` carrying a known code is rejected at encoding (decoding
can never produce one).

#### GEOROUTE (GeoDNS)

Private Scone type (non-IANA range). One geographic routing rule:

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

**Canonical rule**: the endpoints of a given scope are encoded
**sorted** (priority ascending, then weight descending, then target
ascending); sorting is applied at encoding and an unsorted wire is
rejected at decoding (`NonCanonical`), as with record sets. Complete
endpoint duplicates are rejected.

**Validation**: `priority > 0`, `weight ≤ 100` (0 = inactive
endpoint), ≤ 32 endpoints per scope (`GEO_MAX_ENDPOINTS`). A
country/continent code unknown to the local table
(`continent_of_country`) is **accepted** at the wire level: the
resolver's table may be more up to date than the publisher's. The TTL
is that of the parent `RecordData` (unchanged).

### DnsRecord

```text
domain_id[32] sequence.v expiration.v count.v(≤ 256) records…
```

**Canonical rule**: a record set is an unordered collection. It is
encoded **sorted by lexicographic order of each record's individual
encoding**, strictly increasing (duplicates rejected). An unsorted
wire is rejected at decoding (`NonCanonical`). The original `Vec`'s
order is not transported.

### SignedDnsRecord

```text
DnsRecord owner[32] signature(bytes ≤ 128)
```

### RecordHash

```text
RecordHash = BLAKE3-256("SCONE-RECORD-V1" || canonical(DnsRecord))
```

The hash covers the **record content alone**; the owner's signature is
verified separately over the same canonical bytes. *(Decision: the
commitment does not cover the signature — an earlier version of this
document mentioned the "full SignedDnsRecord"; coupling the hash to
the signature brought nothing and tied the commitment to the signature
scheme.)*

## Blocks

### BlockHeader

```text
version.v height.v prev_hash[32] tx_root[32] timestamp.v consensus(bytes ≤ 65,536)
```

- `version`: protocol version (rules above);
- `height`: height (genesis = 0);
- `prev_hash`: hash of the previous block (zeros for the genesis
  block);
- `tx_root`: commitment over the transaction list (Merkle tree defined
  by `scone-blockchain`, see `/docs/en/technical/blockchain.md`);
- `timestamp`: ordering information (Unix, seconds);
- `consensus`: producer payload (`SCONE-BLOCK-V1 ‖ pk[32] ‖
  sig[64]`, see `blockchain.md`) — a network's genesis carries the raw
  `network_id`, and an allocation genesis (mainnet kickstarter) may
  carry an `ALLOC-V1` manifest (≤ 60 KiB); bounded by
  `MAX_CONSENSUS_LEN` (64 KiB).

### Block

```text
BlockHeader tx_count.v(≤ 4096) transactions…
```

Sequential parsing; the counter is capped **before** allocation; any
truncation is rejected. Unlike record sets, the transaction order is
**significant** (consensus order) and preserved exactly.

`BlockHash` (hash of the canonical header) and `tx_root` are computed
by `scone-blockchain` over the encodings defined here (exact formulas
in `/docs/en/technical/blockchain.md`).

## P2P messages

```text
Message = disc u8 || payload
```

Transport (framing, libp2p…) is out of scope; a transport frame should
not exceed `MAX_MESSAGE_LEN` (1 MiB).

| Disc | Message | Direction | Payload | Limit | Expected reply |
|---|---|---|---|---|---|
| `0x01` | `Hello` | both | `version.v` | — | `Hello` |
| `0x02` | `Ping` | both | — | — | `Pong` |
| `0x03` | `Pong` | both | — | — | — |
| `0x04` | `GetBlock` | both | `hash[32]` | — | `Block` |
| `0x05` | `GetBlocks` | both | `start.v` `max.v` | `max ≤ 128` | ≤ `max` × `Block` |
| `0x06` | `Block` | both | `Block` | tx ≤ 4096 | — |
| `0x07` | `Transaction` | both | `Transaction` | — | — |
| `0x08` | `GetRecord` | both | `domain_id[32]` | — | `Record` |
| `0x09` | `Record` | both | `SignedDnsRecord` | — | — |
| `0x0a` | `Checkpoint` | both | `Checkpoint` | signers ≤ 64 | — |
| `0x0b` | `GetCheckpoints` | both | — | — | checkpoint window |
| `0x0c` | `GetStatus` | both | — | — | `NodeStatus` |
| `0x0d` | `NodeStatus` | both | node state (network, tip, height, NAT, domains, mempool, peers, finality, fresh operators) | — | — |
| `0x0e` | `LookupName` | both | `name(str)` | — | `NameState` |
| `0x0f` | `NameState` | both | on-chain state of the name + TLD, incl. the CAS generation (mutations counters, registered_at) | — | — |
| `0x10` | `GetOwnedNames` | both | `owner[32]` | — | `OwnedNames` |
| `0x11` | `OwnedNames` | both | owner's live domains + TLDs | — | — |
| `0x12` | `Error` | reply | application-level error message (`str`) | — | — |

### Checkpoint (M5)

An aggregated checkpoint goes on the wire as: tag `SCONE-CKPT-V1` ||
body `CheckpointData` (116 B, LE — identical to the end of the signing
preimage: `epoch(8) height(8) block_hash(32) prev_checkpoint_hash(32)
state_root(32) recovery(4)`) || `count.v` || `pk[32] ‖ sig[64]` per
signer, **sorted by ascending pk, no duplicates**, `count ≤ 64`
(`MAX_CHECKPOINT_SIGNERS`). Strict decoding: truncation, wrong tag,
non-canonical pk, duplicates or unsorted = rejection (`None`), never a
panic (corruption fuzz tested).

`GetCheckpoints` requests the finality window; the reply is the
`checkpoint_window()` window (at most `CHECKPOINT_KEEP` checkpoints,
oldest to newest) via one or more `Checkpoint` messages.

`TxId` is now defined by `scone-blockchain`
(`BLAKE3-256("SCONE-TX-V1" || canonical(Transaction))`, see
`/docs/en/technical/blockchain.md`). The mempool and windowed
anti-replay (`TxReplay`, `REPLAY_WINDOW_BLOCKS = 256`) are shipped in
the relay (M4); only `GetTransaction` remains deferred (its
definition depends on the remaining consensus decisions).

## Blockchain / DHT split

```text
BLOCKCHAIN                      DHT
──────────                      ───
domain_id                       domain_id
owner                           SignedDnsRecord (full, signed)
sequence
record_hash
```

The chain says *who* owns and *what the current state is* (sequence +
record_hash). The DHT provides *the content*. A DHT record is accepted
if and only if:

1. its signature is valid for the owner registered on the chain;
2. its `sequence` is consistent with (or higher than) the chain's;
3. the `RecordHash` recomputed over its canonical encoding matches the
   chain's `record_hash`.

## Limits

| Constant | Value | Role |
|---|---|---|
| `MAX_NAME_LEN` | 253 | length of a name |
| `MAX_TLD_LEN` | 63 | length of a TLD |
| `MAX_TXT_LEN` | 4096 | TXT string |
| `MAX_PROOF_LEN` | 256 | registration proof |
| `MAX_SIGNATURE_LEN` | 128 | signature (Ed25519 = 64) |
| `MAX_RECORDS_PER_SET` | 256 | records per set |
| `MAX_DHT_RECORD_LEN` | 64 KiB | encoded signed DNS record (DHT payload: wire, storage, gossip) — total bound checked at encoding AND before any decoding; 15–30× above any realistic set (SPF+DKIM+DMARC ≈ 2–5 KiB), 16× under the transport cap. |
| `GEO_MAX_ENDPOINTS` | 32 | endpoints per GeoRoute scope |
| `MAX_UNKNOWN_DATA` | 4096 | data of an unknown type |
| `MAX_TXS_PER_BLOCK` | 4096 | transactions per block |
| `MAX_CONSENSUS_LEN` | 64 KiB | consensus payload of a header (widened for the `ALLOC-V1` genesis manifest — mainnet kickstarter) |
| `MAX_BLOCKS_PER_REQUEST` | 128 | blocks per `GetBlocks` |
| `MAX_MESSAGE_LEN` | 1 MiB | transport frame (recommended) |

Any excess → `LimitExceeded`, detected before allocation: a malicious
peer cannot cause a disproportionate allocation at decoding.

## Canonical encoding

Fundamental rule:

```text
canonical object → canonical binary encoding → hash / signature
```

- minimal varints only;
- record sets sorted strictly increasing;
- a logically equal object has exactly **one** valid encoding;
- decoding rejects non-canonical forms (`NonCanonical`), so there are
  no multiple valid encodings for the same object.

## Error handling

`ProtocolError` (`scone-protocol::error`):

| Variant | Meaning |
|---|---|
| `Truncated` | input ended in the middle of a value |
| `InvalidVarint` | non-minimal, overflowing or too-long varint |
| `IntegerOutOfRange` | integer does not fit its target type |
| `UnsupportedVersion` | zero or higher-than-local version |
| `UnknownDiscriminant` | unknown transaction/message discriminant |
| `LimitExceeded` | length or count beyond a limit |
| `InvalidUtf8` | non-UTF-8 string |
| `TrailingBytes` | bytes remaining after a complete value |
| `Validation` | violation of a `scone-core` invariant |
| `NonCanonical` | per-field-valid but non-canonical encoding |

Decoding never panics on network data.

## Signatures

- scheme: **Ed25519** (RFC 8032) via `scone-crypto`
  (`ed25519-dalek`, `verify_strict` verification — malleable
  signatures and weak keys rejected);
- **chain transactions**: every `RegisterDomain` / `RegisterTld` /
  `UpdateDomain` embeds `public_key` (32 B) + `signature` (64 B) over
  the payload `"SCONE-TX-SIG-V1" || canonical_encode(tx_without_signature)`;
  complete normative specification: `/docs/en/technical/transactions.md`;
- **DHT records** (`SignedDnsRecord`): the signed payload is the
  canonical encoding of the `DnsRecord`; verification crosses
  `record + owner + signature` with the identity registered in the
  blockchain;
- cross-cutting rule: `owner` and all hashes/commitments are
  **recomputed**, never taken from a provided value.

## Consensus rules

Shipped (described in `/docs/en/technical/blockchain.md`): signed
block producer (`SCONE-BLOCK-V1` payload, checkpoint-elected
committee), finality by aggregated checkpoints (`SCONE-CKPT-V1`, fork
choice at the finality floor), registration PoW by name length.
Remaining topics: fine-grained anti-spam fees, owner key rotation.

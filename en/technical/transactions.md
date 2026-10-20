# Signed transactions (format v1)

**Normative** reference document for the signed transaction format.
It specifies the fields, the exact wire order, the sizes, the signed
payload and the validation rules, so as to enable an independent
implementation without reading the Rust. The reference implementation
is [`scone-protocol`](../../../crates/scone-protocol) (encoding) and
[`scone-blockchain`](../../../crates/scone-blockchain) (validation).

`PROTOCOL_VERSION` = **1**. There is only one set of formats: any
version byte other than `0x01` (in a transaction) or different from
the local version (in a block) is rejected with `UnsupportedVersion`.

## Overview

```text
Transaction = disc u8 || version u8 || payload
```

| Field | Size | Role |
|---|---|---|
| `disc` | 1 B | transaction type (table below) |
| `version` | 1 B | **exactly `0x01`** (tx format version) |

**Transactions no longer carry a `network` field on the wire**
(2026-09 change): the `network_id` (`str` ≤ 16, ASCII `[a-z0-9-]` —
e.g. `scone-testnet`, 13 bytes) lives **only in the signing payload**
— it is injected there as the first field by the signer (see [Signed
payload](#signed-payload)) — and stays per-network in the PoW digest.
A transaction **signed** for one network therefore never verifies on
another: its signature fails (`InvalidSignature`) against the payload
recomputed by the chain, which injects its own network (see
`/docs/en/technical/blockchain.md` — networks and genesis).

### Discriminants (normative table)

Each discriminant is a **fixed, arbitrary opaque constant**: the
values are deliberately chosen distinct, so that the wire **never**
suggests a logical order between types (no 0x01/0x02/0x03…
sequence). `0x00` is **never** a valid type (typical corruption byte).
Future types will take any unused free value, documented once and for
all in this table.

| Discriminant | Type | Role |
|---|---|---|
| `0x21` | `RegisterDomain` | claim of a domain |
| `0x52` | `UpdateDomain` | new version of a domain's DNS data |
| `0x93` | `RegisterTld` | claim of a top-level domain (M7) |
| `0x47` | `TransferTld` | transfer of a TLD to a new owner (M8a) |
| `0x6B` | `RevokeTld` | abandonment of a TLD (M8a) |
| `0xB8` | `SetTldOpen` | **irreversible** opening of a TLD to self-registration (one-way, owner-option; `open=false` is a typed rejection) |
| `0xD4` | `AssignDomain` | direct assignment of a domain by the TLD's owner (M8a) |
| `0x3C` | `RenewDomain` | extension of a domain's registration (M8a; carries the canonical name since owner-option — an open TLD follows the expiry) |
| `0x8F` | `RenewTld` | renewal of a PRIVATE TLD's 3-year term (PoW equal to the creation) |
| `0x15` | `SlashTx` | evidence of anchor equivocation + banning (M9) |
| `0xE2` | `TransferDomain` | transfer of a domain to a new owner (M8c) |
| `0x71` | `Heartbeat` | liveness proof of an operator (proof-of-service RFC) |
| `0xA9` | `SetTldOperator` | delegation of a private TLD's operation (proof-of-service RFC) |
| `0x5E` | `SetDomainOperator` | delegation of a domain's operation (proof-of-service RFC) |

Any version byte other than `0x01` is explicitly rejected
(`UnsupportedVersion(received value)`): an unknown or obsolete format
is never parsed by accident.

## SLASH_TX (0x15, M9)

On-chain evidence that an anchor signed two **conflicting checkpoints
at the same epoch** (equivocation). Anyone can submit it — the
evidence is cryptographic and self-contained:

```text
offender[32] | evidence_a (CheckpointData 116 B)
  | sig_a[64] | evidence_b (CheckpointData) | sig_b[64]
  | reporter_pk[32] | reporter_signature[64]
```

- `validate()` checks the structure: same epoch, distinct
  `signing_hash`es, two distinct signatures, reporter signatures over
  the canonical payload;
- state rules (journaled, revertible on reorg): the accused's BOTH
  signatures must verify against the TWO distinct signing hashes at
  the same epoch; the accused must be in the eligible pool
  (`SlashOffenderNotInPool` otherwise); then **banning** — the key is
  removed from the PoS pool persistently (blacklist in state);
- minimal sanction: the accused's domains/TLDs are NOT seized
  (documented M9 decision);
- same checkpoint twice = not equivocation (rejected); different
  epochs = rejected; forged signature = rejected.

## REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

| Field | Type | Constraints |
|---|---|---|
| `name` | `str` ≤ 253 | canonical name of the claimed domain, **carried in cleartext** (see below) |
| `domain_id` | 32 raw bytes | **must** equal `DomainId(name)` (recomputed, never trusted) |
| `owner` | 32 raw bytes | **must** be the derivation of `public_key` (see below) |
| `timestamp` | varint | ordering information (Unix, seconds) |
| `proof` | `bytes` ≤ 256 | registration PoW payload (see [below](#registration-proof-of-work-pow)) |
| `public_key` | 32 raw bytes | signer's Ed25519 public key (RFC 8032) |
| `signature` | **exactly 64 bytes** | Ed25519 signature of the signed payload |

### The name is carried by the transaction

`RegisterDomain` transports the canonical name in cleartext, followed
by its `DomainId`. The consistency `domain_id == DomainId(name)` is
re-checked at encoding, decoding and validation (defense in depth):
a `RegisterDomain` whose id does not match the carried name is
invalid. The `DomainId` remains the consensus identity (chain and DHT
keys); the carried name makes the chain self-descriptive and covers
the entire claim in the signature.

## REGISTER_TLD (0x93)

Claim of a top-level domain (TLD registry, M7):

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) open(0x00|0x01) public_key[32] signature[64]
```

| Field | Type | Constraints |
|---|---|---|
| `tld` | `str` ≤ 63 | canonical name of the TLD **carried in cleartext** (M8b: the PoW challenge derives from the name, just as `RegisterDomain` carries its own) |
| `tld_id` | 32 raw bytes | identity of the TLD: `BLAKE3-256("SCONE-TLD-V1" || tld)` — **must** be the derivation of `tld` (recomputed) |
| `owner` | 32 raw bytes | derivation of `public_key` |
| `timestamp` | varint | ordering information (Unix, seconds) |
| `proof` | `bytes` ≤ 256 | registration PoW payload (see [below](#registration-proof-of-work-pow)) |
| `open` | `0x00\|0x01` | **open** birth: the TLD lands directly in `TldState::Open` (no one, self-service + PoW, a single PoW at the `open_tld_pow_difficulty` rate — cheaper than private: common resource, no PoS seat nor exclusivity); **private** birth: historical behavior (owner, PoS seat, `tld_pow_difficulty`) |
| `public_key` | 32 raw bytes | signer's Ed25519 public key |
| `signature` | **exactly 64 bytes** | Ed25519 signature of the signed payload |

The `TldId` space is **disjoint** from the `DomainId` space (distinct
derivation prefixes — see `/docs/en/general/naming.md`): registering the
TLD `uip` (a claim on the `*.uip` namespace) can never squat a
domain's identity, nor the reverse. State rule at application: the TLD
must be free (registry separate from the domain registry).

## The M8a family — wire formats

Five types complete the TLD registry and the domain lifecycle.
Normative wire formats since M8a; **full state rules since M8b** (see
`/docs/en/technical/blockchain.md`): TLD owner required, PoW wherever a
claim is self-service, expirations and grace on the domain side. No
payload carries a `network` field on the wire (it lives in the signing
payload only).

### TRANSFER_TLD (0x47)

```text
tld_id[32] owner[32] new_owner[32] public_key[32] signature[64]
```

| Field | Type | Constraints |
|---|---|---|
| `tld_id` | 32 raw bytes | the transferred TLD |
| `owner` | 32 raw bytes | derivation of `public_key` (current owner / signer) |
| `new_owner` | 32 raw bytes | recipient's identity (opaque: any value is syntactically valid; binding to the keys that can spend it is an M8b state rule) |
| `public_key` | 32 raw bytes | signer's Ed25519 public key |
| `signature` | **exactly 64 bytes** | Ed25519 signature of the signed payload |

No timestamp or sequence: a TLD has exactly one owner at a time; the
first applied transfer wins (chain order alone).

### REVOKE_TLD (0x6B)

```text
tld_id[32] owner[32] public_key[32] signature[64]
```

Voluntary abandonment of a TLD (re-claimable afterwards by a fresh
`RegisterTld`). Signed by the current owner; deliberately without
timestamp or sequence — a revoke is one-shot against the current
state.

### SET_TLD_OPEN (0xB8)

```text
tld_id[32] owner[32] open u8 open_proof(proof) public_key[32] signature[64]
```

| Field | Type | Constraints |
|---|---|---|
| `open` | 1 byte | **strictly `0x00` or `0x01`** (canonical form; any other value → decoding error, never coercion) |
| `open_proof` | `proof` (length-varint + bytes) | **signed** field; mandatory and verified when `open = 0x01` (opening PoW, network's `open_tld_pow_difficulty`, challenge `SCONE-TLD-OPEN-V1 ‖ tld_id`); empty when `open = 0x00` (closing is free) |

A **closed** TLD = assign-only (domains created exclusively by the
TLD's owner via `AssignDomain`); an **open** TLD = anyone can do the
registration PoW and claim a free domain (`RegisterDomain`).
**Opening** costs an extra proof of work (lighter than the TLD claim:
~3 days on mainnet versus ~1 week), paid once by the owner in
`SetTldOpen` — the proof travels in the signed `open_proof` field and
cannot be substituted after signing.

### ASSIGN_DOMAIN (0xD4)

```text
name(str ≤ 253) domain_id[32] owner[32] assignee[32] public_key[32] signature[64]
```

Mirror of `RegisterDomain`: the canonical name is carried in
cleartext and `domain_id` must equal `DomainId(name)` (recomputed at
decoding, encoding and validation). Signed by the **owner of the
name's TLD** (checked at application, M8b); `assignee` becomes the
domain's first owner (opaque, like `new_owner`).

### RENEW_DOMAIN (0x3C)

```text
name(str ≤ 253) domain_id[32] owner[32] valid_until.v proof(bounded) public_key[32] signature[64]
```

| Field | Type | Constraints |
|---|---|---|
| `name` | `str` ≤ 253 | canonical name **carried in cleartext** (owner-option: the renewal postpones the death of the underlying open TLD, the chain must know the name); `domain_id` must equal `DomainId(name)` (recomputed) |
| `valid_until` | varint | new registration expiry (Unix, seconds) |
| `proof` | bounded | **renewal authorization** (see below) |

Signed by the domain's current owner; whether `valid_until` actually
extends the expiry (and by at most one renewal term) is a state rule
(M8b).

#### Renewal authorization — PoW or PoS depending on the TLD

The cost of a renewal depends on the nature of the underlying TLD
(evaluated at apply time, on the block state):

- **Open TLD**: the renewal costs the **same PoW as the creation** —
  same challenge (`SCONE-DOMAIN-V1 ‖ canonical name`), same
  per-label-length and per-network difficulty, same epoch validity
  window. A domain is never "collected" for free: every term of
  life is paid in work.
- **Private TLD**: the namespace is the TLD owner's stake (PoS).
  The transaction must be **signed by the TLD owner** (or its
  delegated operator) — the signature IS the authorization, the
  `proof` field must be empty (20 zero bytes). The TLD owner
  renews the domains of its namespace under its own rules.

### RENEW_TLD (0x8F)

```text
tld(str ≤ 63) tld_id[32] owner[32] valid_until.v proof(bounded) public_key[32] signature[64]
```

| Field | Type | Constraints |
|---|---|---|
| `tld` | `str` ≤ 63 | canonical TLD name; `tld_id` must equal `TldId(tld)` (recomputed) |
| `valid_until` | varint | must be EXACTLY `now + 3 years` (fixed term — re-derived at apply, any other value rejected) |
| `proof` | bounded | renewal PoW (see below) |

Signed by the TLD's current **private** owner. Apply rules:

- applies to a PRIVATE TLD only (an open TLD lives through its
  domains — typed rejection `TldIsOpen`);
- valid during the term AND through the **60-day grace** that
  follows the expiry; beyond, the TLD is dead (GC) and the
  rejection is `TldExpiredFrozen`;
- **PoW equal to the creation**: challenge
  `SCONE-TLD-V1 ‖ name ‖ current_expiry(BE64)` (the anti-reuse
  suffix binds each PoW to ONE term), same per-length difficulty
  as the initial claim;
- renewal is NOT frozen during the grace — it is the only
  management operation still available to an expired owner.

## UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

| Field | Type | Constraints |
|---|---|---|
| `domain_id` | 32 raw bytes | target domain |
| `owner` | 32 raw bytes | derivation of `public_key` |
| `sequence` | varint | **> 0**, must equal `current + 1` at application |
| `record_hash` | 32 raw bytes | commitment to the DHT record set |
| `public_key` | 32 raw bytes | signer's Ed25519 public key |
| `signature` | **exactly 64 bytes** | Ed25519 signature of the signed payload |

Note: `UpdateDomain` carries **no** `timestamp` — ordering goes
through `sequence`. It also does not carry the name: the domain is
already registered, the id suffices.

### Strict bounds

- `public_key`: **exactly 32 raw bytes**, no length prefix. A pk that
  does not decode as a curve point → `InvalidTransactionKey` error at
  decoding.
- `signature`: **exactly 64 raw bytes**, no length prefix, no margin.
  Any other size is a hard decoding error (truncation rejected).
- `proof`: length-varint + bytes, capped at 256
  (`MAX_PROOF_LEN`), checked before allocation.
- `name`: length-varint + UTF-8 bytes, capped at 253
  (`MAX_NAME_LEN`), revalidated by the `scone-core` naming rules.

## Owner derivation

Hard rule: **the owner is always recomputed, never trusted**.

```text
PublicKeyRef = BLAKE3-256("SCONE-PUBKEY-V1" || public_key)
OwnerId      = BLAKE3-256("SCONE-OWNER-V1" || PublicKeyRef)
```

A transaction whose `owner` does not match the derivation of its own
`public_key` is invalid (`InvalidOwner` on the core side,
`OwnerKeyMismatch` on the chain side), rejected at encoding, decoding
**and** chain validation (defense in depth).

## Signed payload

The signed payload is the canonical encoding of the transaction
**without the signature field**, prefixed by a domain separation tag —
**with the target chain's `network_id` injected at the head**:

```text
signing_payload(tx, network) = "SCONE-TX-SIG-V1"
                              || canonical_encode(tx_without_signature, network)
```

where `canonical_encode(tx_without_signature, network)` follows the
wire structure above, discriminant and version byte included, stopping
before `signature`, **with `network` as the first field** — this is
the ONLY place the network appears. The chain recomputes the payload
with ITS own network: a transaction signed for another network fails
signature verification (`InvalidSignature`); this is the anti
cross-chain replay mechanism.

```text
REGISTER_DOMAIN_no_sig = disc(0x21) version(0x01) network name domain_id[32] owner[32]
                            timestamp.v proof public_key[32]
REGISTER_TLD_no_sig    = disc(0x93) version(0x01) network tld tld_id[32] owner[32]
                            timestamp.v proof public_key[32]
UPDATE_DOMAIN_no_sig   = disc(0x52) version(0x01) network domain_id[32] owner[32]
                            sequence.v record_hash[32] public_key[32]
TRANSFER_TLD_no_sig    = disc(0x47) version(0x01) network tld_id[32] owner[32]
                            new_owner[32] proof public_key[32]
REVOKE_TLD_no_sig      = disc(0x6B) version(0x01) network tld_id[32] owner[32]
                            public_key[32]
SET_TLD_OPEN_no_sig    = disc(0xB8) version(0x01) network tld_id[32] owner[32]
                            open(0x00|0x01) open_proof(proof) public_key[32]
ASSIGN_DOMAIN_no_sig   = disc(0xD4) version(0x01) network name domain_id[32] owner[32]
                            assignee[32] proof public_key[32]
RENEW_DOMAIN_no_sig    = disc(0x3C) version(0x01) network name domain_id[32] owner[32]
                            valid_until.v proof public_key[32]
TRANSFER_DOMAIN_no_sig = disc(0xE2) version(0x01) network domain_id[32] owner[32]
                            new_owner[32] proof public_key[32]
```

Properties:

- deterministic and pure: same transaction ⇒ same payload everywhere;
- the ASCII prefix `SCONE-TX-SIG-V1` (15 bytes,
  hex `53434f4e452d54582d5349472d5631`) prevents any reuse of the
  signature in another hash domain of the protocol;
- all fields except `signature` are covered — **including the name
  carried by a `RegisterDomain`**: modifying any one of them
  invalidates the signature;
- the injected `network_id` is covered: the same unsigned tx signed
  for two networks produces two distinct payloads (hence two distinct
  signatures).

## Registration proof of work (PoW)

The `proof` field of `RegisterDomain` and `RegisterTld` carries the
registration proof of work (`scone-core::pow` module).

### Digest (epoch salt, anti pre-mining)

```text
epoch  = block_height / POW_WINDOW        (POW_WINDOW = 144 blocks)
salt   = BLAKE3-256("SCONE-POW-V1-ARGON2" || network || challenge || epoch_le64)
digest = Argon2id(m, t, p = 1; password = nonce_le64, salt = salt[0..16])
```

`network` is the canonical `network_id` (M8b): a proof mined for one
network is not a proof on another.

`SCONE-POW-V1-ARGON2` also pins the per-network Argon2id parameters
(`pow_params`: mainnet m = 64 MiB, t = 2, p = 1; testnet/devnet
m = 8 KiB, t = 1, p = 1): the Argon2id core (RFC 9106) makes the
cost per hash memory-dominated (GPU/ASIC resistance), and a proof
mined under one parameter set never verifies under another.

`challenge` is the derivation input of the claimed object —
`"SCONE-TLD-V1" || tld` for a TLD, `"SCONE-DOMAIN-V1" || name` for a
domain: exactly the bytes hashed for `TldId` / `DomainId`, so that a
proof is never replayable across namespaces nor across object kinds.
`epoch_le64` and `nonce_le64` are the little-endian 8-byte encodings
of the epoch and the nonce.

The proof is valid when `digest` has at least `difficulty` leading
zero bits.

#### Epoch salt — anti pre-mining

An unsalted digest binds neither time nor height: anyone could pre-mine
proofs today for thousands of names, then register them in a burst at
mainnet launch (squatting) — the proof was self-contained and valid at
whatever block came first. The digest is salted with the
**epoch of the block that APPLIES the transaction**:

- `epoch = block_height / POW_WINDOW`, `POW_WINDOW = 144` blocks
  (~2.6 h at 65 s/block): a grace window for the honest user (mine,
  then submit within the window), mandatory re-mining beyond it;
- the proof DECLARES its epoch on the wire; at application, the
  declared epoch must fall in the applying block's **validity
  window** — BACKWARD tolerance only (late submission is absorbed,
  mining "for the future" is impossible), window sized by
  `epoch_validity_window(difficulty)` = `1 << max(0, difficulty −
  16)` epochs (widened window: high mainnet difficulties mine in
  days-to-weeks, longer than one epoch; stockpiling gains nothing
  because the window ≈ the mining cost — every proof of a burst
  costs its full `2^difficulty` hashes INSIDE the window). At
  difficulty ≤ 16 bits (testnet, long names) the
  strict-equality rule is unchanged;
- mempool admission (best effort): the future block height is
  unknown — the relay verifies against the epoch of
  `current height + 1`; strict application-time verification remains
  the authority;
- deterministic replay: the proof CARRIES its epoch and the window
  is a pure function of (difficulty, block epoch) — a historical
  block re-verifies identically on any node.

Breaking change pre-mainnet: the old compact format (12 bytes) is rejected
with no backward compatibility.

### Wire payload (20 fixed bytes)

```text
nonce_le64 || epoch_le64 || difficulty_le32
```

The difficulty and the epoch travel with the proof, but they are
**producer-declared and therefore never trusted**: the verifier
compares the difficulty against the **protocol constant** for the
registration kind (a payload whose declared difficulty differs from
the constant — lower *or* higher — is rejected, `InvalidProof`), and
the epoch against that of the applying block (`ProofEpochMismatch` on
mismatch). The nonce is then re-verified at the constant difficulty,
digest recomputed from scratch.

### Difficulties (**length-based** curve, M8b + measured calibration)

The difficulty is NOT a flat parameter: it is a deterministic function
of the **created label's length** (`scone-core::pow_length` — the TLD
for `RegisterTld`, the leftmost label for `RegisterDomain`), per
network:

```text
difficulty(len) = apex − round(0.415 × (len − 1)), clamped at the floor
```

| Network | Private TLD (apex/floor) | Born-open TLD / domain (apex/floor) |
|---|---|---|
| testnet | 8 bits / 2 bits | 6 bits / 2 bits |
| mainnet | 42 bits / 30 bits | 40 bits / 30 bits |
| devnet | 2 bits / 1 bit | 1 bit / 1 bit |

A **born-open** TLD (`RegisterTld` with `open = 0x01`) pays the
"opening" curve (apex − 2, like `domain_difficulty`); **opening an
existing private TLD** (`SetTldOpen`) stays at flat difficulty
(`open_tld_pow_difficulty`: 6 bits testnet / 22 bits mainnet /
2 bits devnet — the tx
carries only the `tld_id`, the name paid for its length at creation).
Mainnet benchmarks (private TLD, ~6 M hashes/s): 1 character = 42
bits (≥ 7 days), 5 chars = 40 bits (~2 days), 19 chars = 35 bits
(~1.6 h), 29+ chars = 30-bit floor (~3 min). The curves live in
`scone-core::pow_length` (constants `MAINNET_APEX = 42`,
`MAINNET_FLOOR = 30`, `TESTNET_APEX = 8`, `TESTNET_FLOOR = 2`,
`DEVNET_APEX = 2`, `DEVNET_FLOOR = 1`); the
flat parameters (`tld_pow_difficulty`, etc.) remain in
`scone-core::network::NetworkParams`. Changing a value = a consensus
change **for that network**. Invariant: the devnet is a complete
third `NetworkParams` and testnet stays strictly identical to its
pre-devnet constants (Argon2id parameters, curves, windows) — the
devnet shares NO constant with the other networks beyond the code
itself.

Measured calibration of the flat summits (2026-10, `cargo run
--release -p scone-core --example pow_bench` — single-thread
throughput ~7.7 M H/s on a reference x86-64):

| Kind (mainnet) | difficulty | average real time |
|---|---|---|
| `RegisterTld` | 24 bits | ~7 s |
| `RegisterDomain` | 20 bits | ~54 ms |

At ~65 s/block: a TLD claim costs ~0.1 block of single-core CPU, a
domain ~1 ms of block — symbolic for a user, costly to industrialize
at scale (anti-squat). To recalibrate if hardware targets drop
(mobile ARM: ~4–8× slower).

State branches (M8b, shipped): `RegisterTld` ALWAYS requires its PoW;
`RegisterDomain` requires it on an **open** TLD (the closed path is
assign-only — `AssignDomain`, signed by the TLD's owner, with its own
flat anti-spam proof).

**Cession and assignment proofs (consensus-level anti-spam)**:
`TransferTld`, `TransferDomain` and `AssignDomain` each carry a **flat**
proof of work — `transfer_pow_difficulty` for the two cessions
(challenges `SCONE-TLD-XFER-V1 ‖ tld_id` /
`SCONE-DOMAIN-XFER-V1 ‖ domain_id`), `assign_pow_difficulty` for the
assignment (challenge `SCONE-DOMAIN-ASSIGN-V1 ‖ name`). Same epoch
salt and anti-premining window as registrations; verified at mempool
admission (estimated epoch, best-effort) then at application
(strict). Goal: transfer churn and mass creation under a private TLD
cost consensus-level work, not policy. No per-length taxation: for
assignments, the namespace's exclusivity was already paid when the
private TLD was created.

**Passive cession**: these acquisition paths (`TransferTld`,
`TransferDomain`, `AssignDomain`) are signed by the transferor alone —
the recipient is never consulted. An assumed choice (gifts, dead
drops, bequests), with no imposable PoS debt: see `security-model.md`
(passive acquisition and eligibility).

## Validation rules

A transaction is valid if and only if:

1. **strict decoding**: structure and bounds above respected (known
   discriminant, version `0x01`, pk 32 B, signature 64 B, core
   invariants re-checked at decoding);
2. **owner/pk binding**: `owner == OwnerId::from(public_key)`
   (recomputed);
3. **name/id consistency** (RegisterDomain): `domain_id ==
   DomainId(name)` (recomputed);
4. **signature**: Ed25519 `verify_strict` of `signature` against the
   signed payload **recomputed from scratch** with the chain's network
   (never taken from a peer or from the transaction as provided) — a
   tx signed for another network fails here (`InvalidSignature`,
   anti cross-chain replay);
5. **state rules** (at application in a block — M8b, see
   `/docs/en/technical/blockchain.md`):
   RegisterTld → TLD free + not ICANN + PoW (length-based
   difficulty); RegisterDomain → TLD registered **and open** + free
   domain (grace included) + PoW (length-based);
   UpdateDomain → existing domain, current `owner`,
   `sequence == current + 1`;
   TransferTld / RevokeTld / SetTldOpen / AssignDomain → existing
   **and private** TLD, signer = the TLD's owner;
   TransferDomain → existing domain, owner;
   RenewDomain → existing domain, strict extension ≤ 3 years,
   PoW authorization (open TLD: same PoW as the creation,
   **bound to the current expiry** — a challenge distinct from
   the registration and every earlier renewal) or PoS
   (private TLD: TLD owner's — or its effective operator's —
   signature, empty `proof`).

Verification uses `verify_strict` (rejection of malleable signatures
and weak/non-canonical keys), as required for untrusted data.

## Operation order in `push_block`

Cryptographic validation (2 to 4) takes place for each transaction
**before** state application, on an atomic working state per block;
any error leaves the chain rigorously unchanged.

## Block assembly

`BlockBuilder` (`scone-blockchain`) assembles a block from a
transaction queue: `tx_root` recomputed over the ordered list, header
chained onto the parent hash (`prev_hash`), `version` =
`PROTOCOL_VERSION`. The consensus payload (signed producer payload)
and the timestamp are provided by the caller; the existing `Consensus` trait
is used symmetrically with validation.

## Hex examples (real, reproducible vectors — M8b)

### RegisterDomain `example.uip`, timestamp 42, testnet

Signed payload (pinned vector; historically generated by the
removed offline tooling — the values remain valid vectors). The `network`
appears here because this is the SIGNING PAYLOAD (its only place);
the signed wire transaction itself contains none:

```text
53434f4e452d54582d5349472d5631       SCONE-TX-SIG-V1 prefix
21                                    disc REGISTER_DOMAIN (0x21)
01                                    format version
0d 73636f6e652d746573746e6574         network (str, "scone-testnet")
0b 6578616d706c652e756970             name (str, length 11)
e6a2cbe264f81ed7da4d901c4deec6a3579224b8080fad01cb7461b8c264f181  domain_id
7b2446dadddf640244ac8677a401f148b2b23d1c836d5ad3dd340f8ef2edbed8   owner (fixed build key)
2a                                    timestamp (varint, 42)
14 0200000000000000 0000000000000000 04000000
                                      proof (20 B: nonce 2, epoch 0, difficulty 4 — testnet)
2a002152f8d19b791d24453242e15f2eab6cb7cffa7b6a5ed30097960e069881db12  pk + start of sig
```

### RegisterTld `uip`, timestamp 42

Signed payload (pinned vector) — the `TldId` is the derivation vector
pinned in `/docs/en/general/naming.md`:

```text
53434f4e452d54582d5349472d5631       SCONE-TX-SIG-V1 prefix
93 01                                 disc REGISTER_TLD + version
0d 73636f6e652d746573746e6574         network (str, "scone-testnet")
03 756970                             tld (str, "uip")
ad5a86d68643d5c22d6a959bb1a315530c77dfda241f1ac1a8107450f3fab25e  tld_id ("uip")
7b2446dadddf640244ac8677a401f148b2b23d1c836d5ad3dd340f8ef2edbed8   owner
2a                                    timestamp (varint, 42)
14 8e00000000000000 0000000000000000 07000000
                                      proof (20 B: nonce 0x8e, epoch 0, difficulty 7 — testnet, 3-char label)
00                                    open (0x00 = private birth)
2a002152f8d19b791d24453242e15f2eab6cb7cffa7b6a5ed30097960e069881db12  pk
```

(The full values were reproducible via the removed offline
tooling; the vectors below remain pinned and valid. The PoW is
mined automatically at network difficulty — nonces vary from run
to run; only network/name/id/owner/pk are stable. Since the
2026-09 wire change, the hex of the signed transaction NO LONGER
contains the bytes
`0d 73636f6e652d746573746e6574`: the network travels only in the
payload to be signed.)

## Where the hex lives now

The offline `tx build|sign|verify` commands were removed
(cli-simplify-v1). The signed-hex format itself is unchanged and
remains the wire format of the **web/API flow**: the registrar
web app builds, signs (in-browser keystore) and submits
transactions as canonical hex over the API. On the CLI side,
the user paths are the one-command forms:

```bash
scone domain register example.uip        # mines, signs, submits
scone domain renew example.uip           # renewal proof, signs, submits
scone domains                            # what you own
```

In all code paths the owner and key are **recomputed** from the
signing identity, never taken from any input.

## Version history

| Version | Contents | Status |
|---|---|---|
| 1 | embedded `public_key` + `signature`, payload `SCONE-TX-SIG-V1`, version byte `0x01`, `RegisterDomain` carries the name, `RegisterTld` (0x93), M8a family: `TransferTld` (0x47), `RevokeTld` (0x6B), `SetTldOpen` (0xB8), `AssignDomain` (0xD4), `RenewDomain` (0x3C) — normative wire formats; M8b: `network` field signed at the head of every payload, `RegisterTld` carries the name, full state rules, per-network PoW difficulties | superseded |
| 1 (2026-09, wire without network) | **removal of the `network` field from the wire of all transactions**: the network_id lives only in the signing payload (`signing_payload(tx, network)` injects it at the head); network separation relies on the signature (`InvalidSignature` on a foreign payload, the `WrongNetwork` error disappears); `TX_FORMAT_VERSION` stays `0x01` (relaunchable testnet, no coexistence); **all TxIds change** (the TxId is the hash of the signed wire encoding, which loses 14–15 bytes); PoW digest, genesis and per-network data-dirs unchanged | superseded |
| 1 (2026-10, PoW epoch salt) | **anti pre-mining PoW digest**: salt `BLAKE3("SCONE-POW-V1-ARGON2" ‖ network ‖ challenge ‖ epoch_le)` + Argon2id core over `nonce_le`, with `epoch = block_height / 144` of the block that APPLIES the tx; proof wire payload 12 → **20 bytes** (`nonce_le ‖ epoch_le ‖ difficulty_le`), the declared epoch is verified against the applying block's epoch validity window (typed rejection `ProofEpochMismatch`), best-effort `(height+1)/144` at mempool admission, `--epoch` CLI flag on offline paths; breaking change pre-mainnet: the old 12-byte payload is rejected, **all TxIds change** (the `proof` field grows by 8 bytes) | current |

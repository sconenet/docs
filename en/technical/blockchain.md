# Blockchain

Reference document for the blockchain layer: identities, hashing, genesis,
canonical state, transaction ordering, and block validation.
The reference implementation is the crate
[`scone-blockchain`](../../../crates/scone-blockchain), on top of
`scone-protocol` (canonical encoding), `scone-core` (business rules), and
`scone-crypto` (BLAKE3).

Current scope: blockchain logic with real consensus (signed producer,
checkpoints, finality), **in-memory state** on top of a KV backend
(concrete storage lives in `scone-storage`) — see
[Deferred to consensus](#deferred-to-consensus).

## Identities and hash domain separation

Four identities, four roles, four distinct BLAKE3 prefixes — a prefix is
never reused across two usages:

| Type | Role | Prefix | Formula |
|---|---|---|---|
| `DomainId` | domain name identity | `SCONE-DOMAIN-V1` | see `/docs/en/general/naming.md` |
| `RecordHash` | DNS content identity | `SCONE-RECORD-V1` | see `/docs/en/technical/protocol.md` |
| `TxId` | transaction identity | `SCONE-TX-V1` | below |
| `BlockHash` | block identity | `SCONE-BLOCK-V1` | below |

The Merkle tree uses its own prefix `SCONE-MERKLE-V1`. These prefixes are
in addition to those of `scone-core` (`SCONE-PUBKEY-V1`,
`SCONE-OWNER-V1`).

Full dependency chain:

```text
DomainId
    ↓
RegisterDomain / UpdateDomain
    ↓
Transaction
    ↓
TxId
    ↓
MerkleRoot
    ↓
BlockHeader
    ↓
BlockHash
    ↓
prev_hash of the next block
    ↓
canonical chain
```

## TxId

```text
TxId = BLAKE3-256("SCONE-TX-V1" || canonical_encode(Transaction))
```

- deterministic: the same transaction (same canonical encoding) ⇒ same
  `TxId`, on all nodes, forever;
- identity **by content**: any modified field changes the `TxId`
  (domain, owner, timestamp, proof, sequence, record_hash);
- no local timestamp or non-canonical value enters the computation;
- no dependency on storage or network;
- the identity does not guarantee uniqueness of application: replaying an
  identical transaction fails on state rules (double `RegisterDomain`,
  sequence replay), not on the `TxId`.

API: `scone_blockchain::transaction_id(&Transaction) -> Result<TxId>`.

## Merkle tree (tx_root)

Commitment over the **ordered list** of a block's transactions:

```text
0 transactions  root = BLAKE3-256("SCONE-MERKLE-V1" || "EMPTY")
leaf            BLAKE3-256("SCONE-MERKLE-V1" || 0x00 || TxId)
internal node   BLAKE3-256("SCONE-MERKLE-V1" || 0x01 || left || right)
odd level       the last hash is duplicated to complete the pair
```

Rules:

- **never sorted**: the order of transactions in a block is
  significant; two blocks containing the same transactions in a
  different order have different roots;
- the `0x00` / `0x01` tags prevent any confusion between a leaf and an
  internal node (second-preimage);
- the `SCONE-MERKLE-V1` prefix isolates the tree from the other hash
  domains (a `TxId` cannot be interpreted as a node);
- in-house implementation (~30 lines), deterministic and tested; no
  external Merkle library.

API: `merkle_root(&[TxId]) -> MerkleRoot` and
`tx_root(&[Transaction]) -> Result<MerkleRoot>`.

## BlockHash

```text
BlockHash = BLAKE3-256("SCONE-BLOCK-V1" || canonical_encode(BlockHeader))
```

The hash covers **only** the canonical header: the transactions are
already committed by `tx_root`, so they are not hashed a second time:

```text
transactions -> TxIds -> MerkleRoot -> BlockHeader -> BlockHash
```

Modifying a transaction, their order, the height, the timestamp, the
`prev_hash`, or the consensus payload changes the `BlockHash`.

API: `block_hash(&BlockHeader) -> Result<BlockHash>`.

## Networks and genesis (M8b)

Scone defines three disjoint networks, identified by a canonical
`network_id` (1..=16 ASCII bytes `[a-z0-9-]`):

| Network | `network_id` | private TLD apex | open TLD apex | domain apex | TLD floor | domain floor |
|---|---|---|---|---|---|---|
| testnet | `scone-testnet` | 8 bits | 7 bits | 6 bits | 2 bits | 1 bit |
| mainnet | `scone-mainnet` | 25 bits | 24 bits | 23 bits | 22 bits | 19 bits |
| devnet | `scone-devnet` | 2 bits | 1 bit | 1 bit | 1 bit | 1 bit |

The parameters live in `scone-core::network::NetworkParams`
(named instances `TESTNET` / `MAINNET` / `DEVNET`, lookup via
`NetworkParams::by_name`). Changing a difficulty = a consensus change
**for that network**.

**Structural invariant**: the devnet is a complete third
`NetworkParams` — never a subset of testnet flags. Testnet stays
**strictly identical** to what it was before the devnet was added
(same Argon2id constants, same curves, same epoch windows); the test
suite pins this invariant.

**Length-variable difficulty** (`scone-core::pow_length`): the
difficulty of a creation depends on the length of the **created label**
(the TLD for `RegisterTld` — private or newly opened —, the leftmost
label for `RegisterDomain`). The formula yields:

```text
TLD     : difficulty(len) = apex − round(0.287 × (len − 1)), clamped at the floor
domain  : difficulty(len) = apex − round(0.480 × (len − 1)), clamped at the floor
```

Two distinct slopes: `log₂(1.22) ≈ 0.287` bit/character for TLDs
(each extra character divides the cost by ~1.22), and
`log₂(1.40) ≈ 0.480` bit/character for domains — domains decay
faster: their value sits in the TLD carrying them. A born-open TLD
sits at the private apex − 1; a domain at the private apex − 2
(mainnet: private TLD 25, open TLD 24, domain 23). Mainnet
benchmarks (memory-hard Argon2id, ~11.4 H/s single-thread):
private TLD 1 char = 25 bits ≈ 34 days, 9+ chars = 22-bit floor
≈ 4.3 days; open TLD 1 char = 24 bits ≈ 17 days; domain 1 char =
23 bits ≈ 8.5 days, 9+ chars = 19-bit floor ≈ 13 h. **Opening** an
existing private TLD (`SetTldOpen`) stays at flat difficulty
(`open_tld_pow_difficulty`: 6 bits testnet / 22 bits mainnet): the tx
carries only the `tld_id`, the name paid its length at creation.
Pure function (determinism: two nodes compute the same difficulty
for the same name).

Genesis is a protocol constant **per network**, reproducible from the
constants alone (no randomness, no local clock, no network):

| Field | Value |
|---|---|
| `version` | `PROTOCOL_VERSION` (`1`) |
| `height` | `0` |
| `prev_hash` | 32 zero bytes (no previous block) |
| `tx_root` | Merkle root of the empty list |
| `timestamp` | `GENESIS_TIMESTAMP = 0` (purely structural) |
| `consensus` | the canonical bytes of the `network_id` |
| transactions | empty |

**Allocation genesis** variant (mainnet kickstarter, M-next): the
`consensus` field carries `network_id ‖ ALLOC-V1 ‖ count.v ‖ entries`
(pre-paid domains/TLDs, canonical sort by name, manifest ≤ 60 KiB,
≤ 2048 allocations) — this widened format is what justifies
`MAX_CONSENSUS_LEN` = 64 KiB (see `scone-blockchain::genesis_alloc`).

The `network_id` in the consensus payload makes the genesis hashes —
and therefore the entire blockchain — **disjoint across networks**: a
testnet block can never attach to a mainnet chain
(`UnknownParent`), no replay is possible without re-signing
everything.

API: `genesis_of(&NetworkParams) -> Block`,
`genesis_hash_of(NetworkId) -> BlockHash`; the aliases `genesis()` /
`genesis_hash()` refer to testnet (the development default).

### Devnet — resettable local network

The devnet (`scone-devnet`) is the third network, dedicated to fast
local iteration:

- **Own genesis**: the `scone-devnet` network id in the consensus
  payload yields a genesis hash distinct from BOTH testnet and
  mainnet — no genesis, block, signature or PoW proof ever crosses
  the boundary (replay is impossible).
- **Near-zero PoW**: curve pinned in the 0..=2 bit range
  (`DEVNET_APEX = 2`, `DEVNET_FLOOR = 1` — the length decay still
  applies): a TLD or domain claim mines in milliseconds, exercising
  exactly the same code path as the other networks.
- **Fast PoS consensus**: same parameters as testnet (committee of
  4, 10 s epochs) for quick finality on a local mesh.
- **Reset**: the chain lives in its own data directory
  (`~/.scone/devnet/` by default). **Wiping that directory and
  relaunching the relay resets the chain** (fresh genesis, virgin
  store) — no extra tooling needed. NEVER point a devnet data dir at
  another network's data dir: a non-empty store holding another
  chain is refused (`Corrupted`).

## Canonical state

```text
DomainState  = { owner: OwnerId, sequence: u64, record_hash: Option<RecordHash>,
                 registered_at: u64, valid_until: u64 }
TldState     = enum { Private { owner: OwnerId } | Open }
               -- enum (owner-option): the open/private invariant is
               -- structural; the death deadline lives outside the entry
               -- (tld_floors table + journaled death schedule)
ChainState   = NetworkParams                  (bound network, M8b)
               + DomainId -> DomainState      (direct access, in memory)
               + TldId -> TldState            (TLD registry, M7b)
               + DomainId -> (expired_at, OwnerId)   (grace windows, M8b)
```

* `registered_at` / `valid_until` (M8b): registration expiry,
  evaluated against the **parent block's timestamp** (deterministic,
  committed in the header — never the local clock);
* `open`: a private TLD can be **born open** (`RegisterTld` with
  `open = 0x01` — no one, self-service) or **closed** at claim time
  (closed = assign-only), then become open via `SetTldOpen`
  (self-registration with PoW) — **irreversibly**;

- direct access by `DomainId` (32 bytes): never a `String` as key,
  never a full scan — a necessary condition to target hundreds of
  billions of domains;
- `record_hash == None` as long as no `UpdateDomain` has been applied;
- the TLD and domain id spaces are **disjoint by construction**
  (distinct derivation prefixes, see `/docs/en/general/naming.md`):
  registering the TLD `uip` can never squat a domain's identity, nor
  the other way around.

### Application rules

`state.apply(transaction)` is deterministic and atomic (on error, the
state is unchanged):

**All transactions (M8b, wire 2026-09)**: transactions no longer
carry a `network` field — separation relies on the **signature**: the
signed payload injects the chain's `network_id`
(`validate_transaction(tx, network_of_the_chain)`), so a tx signed
for another network fails signature verification
(`InvalidSignature`, BEFORE application — a testnet-signed tx is
never applied, never pooled by a mainnet node, and vice versa).

**REGISTER_TLD**:

- the TLD must be **free** (absent from the TLD registry) and **not
  ICANN-reserved** (`IcannTldReserved`: ICANN root TLDs belong to no
  one on Scone);
- **mandatory PoW**: `pow::verify` on the challenge
  `SCONE-TLD-V1 || tld` (derived from the name carried by the tx), at
  the **length-based** difficulty of the label (`pow_length`):
  private birth → `tld_difficulty` (network apex), open birth
  (`open = 0x01`) → `open_tld_difficulty` (apex − 2) — the digest is
  recomputed from scratch
  (salt `SCONE-POW-V1-ARGON2 ‖ network ‖ challenge ‖ epoch`,
  Argon2id core), the
  self-declared difficulty must equal the expected difficulty, and
  the **epoch declared** by the proof must equal
  `height/POW_WINDOW` of the applying block (epoch salt against
  pre-mining — typed rejection `ProofEpochMismatch`, see
  `transactions.md`);
- on application: `open = 0x01` → `{ Open }` (no one — self-service,
  PoS seat released, death floor laid at `now + grace`);
  `open = 0x00` → `{ Private { owner } }` — **closed by default**
  (assign-only); opening is explicit via `SetTldOpen`.

**REGISTER (RegisterDomain)**:

- **the TLD of the carried name must be registered AND open**: the
  chain derives `TldId(name.tld())` itself (never a supplied value);
  absent → `UnknownTld` (D1, M7c), present but closed → `TldClosed`
  (M8b: the assign-only path is exclusive);
- the domain must be **free**: absent from the state AND outside any
  grace window of an expired registration (see GC below);
- **mandatory PoW**: challenge `SCONE-DOMAIN-V1 || name` at the
  **length-based difficulty of the leftmost label**
  (`pow_length::domain_difficulty` — the TLD being open, this is the
  self-service entry price);
- on application: `{ owner, sequence: 0, record_hash: None,
  registered_at: now, valid_until: now + 1 year }` — **registration
  lasts 1 year** (365 days).

**Intra-block order**: significant. `[RegisterTld, SetTldOpen,
RegisterDomain]` within the same block is valid; any permutation that
applies a rule before its precondition exists is rejected as a whole
block (undo-log).

**UPDATE (UpdateDomain)**:

- the domain must exist (not expired);
- `tx.owner` must be the current owner;
- `tx.sequence` must be **exactly** `current_sequence + 1`
  (no gaps, no replay);
- on application: `sequence` advances, `record_hash` is replaced.

**TRANSFER_TLD (0x47)**:

- the TLD must exist and be **private** (`UnknownTld` / `TldIsOpen`
  otherwise — an open TLD belongs to no one);
- the signer must be the current owner (`NotTldOwner` otherwise);
- on application: `owner = new_owner` (the recipient's identity is
  opaque; the binding to the keys that can spend it is done by
  subsequent txs). No sequence: the first applied transfer wins
  (chain order).

**REVOKE_TLD (0x6B)**:

- the TLD must exist and be **private** (an open TLD cannot be
  revoked — `TldIsOpen`: it dies naturally at GC), signer = current
  owner;
- on application: the TLD is removed from the registry and becomes
  re-claimable by a fresh `RegisterTld` (PoW included).

**SET_TLD_OPEN (0xB8)**:

- the TLD must exist and be **private** (already open → `TldIsOpen`:
  the former owner is erased, no legitimate key exists anymore to
  sign for it); `open = 0x00` (closing) → typed rejection
  `TldCloseUnsupported` as early as pure validation: **opening is
  irreversible** (owner-option);
- signer = current owner (`NotTldOwner`);
- **opening = mandatory PoW**: `pow::verify` on the challenge
  `SCONE-TLD-OPEN-V1 || tld_id` at the network's flat
  `open_tld_pow_difficulty` (6 bits testnet / 22 bits mainnet),
  epoch salt included (the proof travels in the tx's signed
  `open_proof` field);
- on application: `TldState::Open` — **the owner is erased**
  (irreversible, PoS seat released), a death floor is laid at
  `now + grace`.

**ASSIGN_DOMAIN (0xD4)**:

- the TLD of the carried name must exist and be **private**
  (`TldIsOpen` otherwise: an open TLD is self-service only), signer =
  owner of the TLD (`NotTldOwner`);
- the domain must be free (same grace rule as REGISTER);
- on application: `{ owner: assignee, sequence: 0, record_hash:
  None, registered_at: now, valid_until: now + 1 year }`. **No
  PoW**: it is the namespace owner who vouches.

**TRANSFER_DOMAIN (0xE2, M8c)**:

- the domain must exist (`UnknownDomain` otherwise), signer = current
  owner (`NotOwner`);
- on application: `owner = new_owner` (the PoS seat follows the
  domain in the owner pool, symmetric with `TransferTld`). No
  sequence: the first applied transfer wins (chain order).

**RENEW_DOMAIN (0x3C)**:

- the domain must exist; **the TLD kind is evaluated FIRST**
  (M8c hardening):
  - **private TLD**: **PoS** authorization — the signer IS the
    TLD owner, or the TLD owner's seat delegates to the signer's
    public key
    (`effective_operator(tld_owner) == renew.public_key` — the
    delegation is always read FROM the TLD owner, never the
    other way). The `proof` field must be empty (a non-empty
    proof = reject).
  - **open TLD**: **PoW** authorization — the signer is the
    current domain owner (`NotOwner` otherwise) and the proof
    mines the renewal challenge: same difficulty as the creation
    (by label length), **bound to the current expiry**
    (`challenge = DOMAIN_ID_VERSION ‖ canonical name ‖ current
    valid_until`) — a renewal's proof is distinct from the
    registration's and from every earlier renewal's
    (anti-reuse);
- `valid_until` must **strictly extend** the current deadline
  (`RenewalNotExtending` otherwise);
- `valid_until ≤ now + 3 years` (`RenewalExceedsTerm` —
  anti-hoarding bound: **1 renewal term = 1 year, 3 years ahead
  maximum**);
- on application: `valid_until = tx.valid_until`; the renewal also
  postpones the death floor of the underlying open TLD (the tx
  carries the canonical name for that purpose).

**Deterministic GC (M8b)**: before applying a block's transactions,
the chain evaluates expirations at the **parent block's timestamp**:
any domain whose `valid_until ≤ now` is removed from the live state
and parked in a **30-day grace window**
(`valid_until + 30 days`). During the grace period, only the former
owner can re-claim the name; afterwards, it is free for everyone.
The GC is journaled: a rejected block restores the pre-GC state bit
for bit.

**TLD lifecycle (fixed terms)**: TLDs carry their own clock,
independent of the domains.

- **Private TLD**: the claim opens a **3-year term**
  (`TLD_TERM_SECS`). Renewal (`RenewTld`, 0x8F) pays a **PoW equal
  to the creation** and resets exactly 3 years (fixed term
  re-derived at apply). At expiry a **60-day grace**
  (`TLD_GRACE_SECS`) opens: renewal stays possible, but **every
  other management operation is frozen** — notably `AssignDomain`
  (typed rejection `TldExpiredFrozen`): an expired owner loses
  management rights until they renew. Domains already assigned
  underneath live their own `valid_until` — the TLD freezes, not
  its children. Grace elapsed without renewal: the TLD dies at GC
  (PoS seat released, re-claimable). A `TransferTld` carries the
  remaining term.
- **Open TLD**: born (or opened) with a **1-year floor**
  (`OPEN_TLD_FLOOR_SECS` — anti instant create-destroy). Every
  living domain underneath pushes the death clock back (it follows
  the most recent domain's expiry). No domain left + floor
  elapsed: death at GC, re-claimable.
- **Immortal open TLD** (genesis only, e.g. `.scone`): the
  `OpenImmortalTld` allocation sets the `until = u64::MAX`
  sentinel — the GC can never cross it and the monotone re-arm
  can never lower it: the TLD never expires, even emptied of
  all its domains. No runtime transaction can create an
  immortal (`RegisterTld` keeps its 1-year floor); domains
  underneath live their normal cycle.

TLD deaths share the domains' deterministic death queue
(`tld_deaths`), journaled and bit-exactly restorable.

**Height-1 exception**: genesis carries a STRUCTURALLY null timestamp
(`GENESIS_TIMESTAMP` = 0, a protocol convention — not a clock). The
chain time of the FIRST block is therefore its own timestamp
(committed in its header, deterministic), not its parent's: anchoring
block 1's expirations at 0 would pin the death floor of a TLD born
open (and the `valid_until` of a domain born at block 1) to the Unix
epoch — the pre-apply GC of block 2 (real timestamp) would remove the
resource before application ("dropped conflicting tx from block
production: unknown TLD", testnet regression 60).

**Replay**: cross-network replay is impossible at three levels —
disjoint genesises (`UnknownParent`), `network_id` injected into
every signature payload (invalid signature on the other network —
the only place the network appears since wire 2026-09), PoW separated
per network in the digest. Intra-network replay of an identical tx
fails on state rules (double claim, sequence), never on the `TxId`.

Two nodes of the same network starting from the same state and
applying the same blocks in the same order produce exactly the same
final state.

## Transaction ordering

Order is a fundamental property, committed at every level:

```text
transaction order in the block
    ↓
MerkleRoot
    ↓
BlockHash
    ↓
canonical chain
```

Two transactions are not interchangeable: within the same block,
`[REGISTER, UPDATE]` applies, `[UPDATE, REGISTER]` is rejected
(domain nonexistent at the time of the `UPDATE`). The global
tie-break (which orders the mempool, which produces the block)
belongs to consensus.

## Block validation (push_block)

Everything is **recomputed**, nothing is taken on faith — a `tx_root`,
hash, signature, or field supplied by a peer is never taken at face
value:

1. `prev_hash` == hash of the canonical tip (otherwise `UnknownParent`
   if the parent is unknown, `ParentNotTip` if it is known but not the
   tip: minimal fork detection);
2. `height` == tip height + 1;
3. `version` == `PROTOCOL_VERSION` **exactly** (any other version —
   lower or higher — is rejected, not reinterpreted);
4. transaction count ≤ `MAX_TXS_PER_BLOCK` (4096);
5. `tx_root` recomputed over the transactions in block order;
6. consensus hooks (`validate_header`, `validate_tx`);
6ter. **windowed TXID anti-replay (.bak port)**: a transaction whose
   `TxId` appears in the inclusion index of the
   `REPLAY_WINDOW_BLOCKS = 256` most recent blocks is rejected
   (`TxReplay`). The index (TxId → inclusion height) is a pure
   function of the canonical chain, pruned at the same pace; the
   index is rebuilt at boot by storage (rescan of the last
   `min(256, height)` persisted blocks — M8, see
   `/docs/en/technical/storage.md`): a restarted node rejects the same
   replays as a live node. Beyond the window, re-inclusion remains
   subject to state rules (double claim, sequence replay).
6bis. **signed producer (M5, .bak port)**: the `consensus` payload
   must decode as a producer payload
   (`SCONE-BLOCK-V1 ‖ pk[32] ‖ sig[64]`, bounded); the signature
   covers the *producer signature hash* = the header's `BlockHash`
   **with the `consensus` field emptied** (the payload lives in that
   field; signing the full hash would be circular); the producer must
   belong to the set authorized at the block's `timestamp` (committee
   elected by checkpoint, plus unlocked recovery draws; bootstrap:
   owners of live domains — an empty pool = open production,
   otherwise the first registration would be impossible). Genesis
   (height 0, empty payload) is structural and never concerned. Typed
   error `InvalidProducer`;
7. for each transaction, in order: **cryptographic** validation
   (`validate_transaction`: owner/key binding recomputed +
   `verify_strict` on the signed payload recomputed from scratch, see
   `/docs/en/technical/transactions.md`), then application to the
   canonical state under an **undo journal** (undo-log: one entry per
   applied transaction, cost O(transactions of the block), not
   O(domains));
8. atomic commit: everything passes ⇒ block added, journal discarded;
   the slightest error ⇒ journal rewind (reverse order) and the chain
   rigorously unchanged, bit for bit.

No validation function panics; any malformed or hostile input
produces a typed `BlockchainError`
(`OwnerKeyMismatch`, `InvalidSignature`, …).

## Block assembly (BlockBuilder)

`BlockBuilder` (`scone-blockchain::builder`) assembles a block from a
queue of validated transactions:

```text
BlockBuilder::after(parent_height, prev_hash)
    .with_timestamp(t) .with_consensus(payload)
    .push_tx(tx)… .build() -> Block
```

- `tx_root` **recomputed** over the ordered list of queued
  transactions (never supplied);
- `prev_hash`/`height` chained onto the parent (the parent hash is
  computed by the caller via `block_hash`, never taken from a peer);
- `version` = `PROTOCOL_VERSION`;
- the `MAX_TXS_PER_BLOCK` bound enforced at insertion
  (`TooManyTransactions`);
- the `consensus` payload (signed producer payload) and the `timestamp` are
  supplied by the caller: concrete consensus plugs in via the existing
  [`Consensus`](#consensus-abstraction) trait, symmetrically to
  validation;
- M5: `.with_producer(&SigningKey)` signs the block as producer —
  the `consensus` payload becomes the signed producer payload (over
  the producer signature hash defined above) and replaces any prior
  `.with_consensus`.

An assembled block passes `push_block` without re-encoding (tested).

## Forks and fork choice (M3c, .bak port)

`push_block` remains the linear entry point (parent == tip, otherwise
`UnknownParent`/`ParentNotTip`). `try_attach` (M3c) evaluates
competing blocks:

- unknown parent → `UnknownParent` (the block waits);
- valid lineage but lagging → branch evaluation: the longest chain
  wins; perfect tie → deterministic tie-break (smallest tip hash in
  lexicographic order — two nodes holding the same blocks always pick
  the same winner, tested);
- **finality floor**: a branch one of whose ancestors contradicts a
  finalized checkpoint (included in the `prev_checkpoint_hash` of an
  accepted checkpoint) is refused (typed error
  `Consensus("branch contradicts a finalized checkpoint")`);
- adoption = replay from genesis of the winning branch's blocks
  (bit-exact determinism tested: two nodes receiving the same blocks
  in different orders end up in the same state);
- no received block is canonical before full validation.

## Bounded RAM window (M7a, .bak port)

The chain keeps only the **last 512 blocks** in RAM
(`RAM_WINDOW_BLOCKS`, port of the .bak's `HISTORY_KEEP = 512`), above
the **finality floor** (the last finalized checkpoint):

- a block leaves RAM only if it is **both** outside the window
  (`height < tip − 511`) **and** strictly below the finality floor —
  a reorg above the floor always remains possible, so those blocks
  are NEVER evicted;
- before the first finalized checkpoint (bootstrap), **nothing is
  evicted**: bounded memory starts with finality;
- eviction is real: blocks, the hash index (`known_hashes`, itself
  bounded, indexed by height), and txs leave memory — the store
  (`NodeStore`, `scone-storage`) holds the durable copies;
- `block(h)` below the window → `None`; `block_result(h)` → typed
  error `BlockPruned { height }` — the reorg paths
  (`push_block`, `try_attach`, `adopt_branch`, prefix replay)
  propagate it: the relay must reload the segment from storage or fall
  back to a full sync (documented limitation: no segmented adoption
  before the full relay port);
- a block built on an evicted parent classifies as `BlockPruned`
  (not `UnknownParent`): parent height < `base_height` ⇒ the store can
  serve it.

## Consensus abstraction

```rust
pub trait Consensus {
    fn validate_header(&self, header: &BlockHeader) -> Result<()>;
    fn validate_tx(&self, tx: &Transaction) -> Result<()>;
}
```

- `PermissiveConsensus` (default): accepts everything — used until the
  real rules are defined;
- the hooks are called by `push_block`: concrete consensus (PoW,
  difficulty, fees, timestamp/anti-replay, fork choice) will plug in
  without rewriting the chain;
- a single abstraction, no trait hierarchy.

## Scalable state (KV backend, SMT pivot, expiry queue, bounded owner pool)

Goal: 250+ billion potential domains — NEVER load everything into
RAM. Four pieces:

### KV backend (`state_backend.rs`)

```rust
pub trait StateBackend: Default + Send {
    fn get(&self, key: &Pkey) -> Option<Vec<u8>>;
    fn put(&mut self, key: Pkey, value: Vec<u8>);
    fn delete(&mut self, key: &Pkey);
    fn range(&self, from: Option<&Pkey>, limit: usize)
        -> (Vec<(Pkey, Vec<u8>)>, Option<Pkey>);
}
```

- `Pkey` = the 32 raw bytes of the `DomainId`/`TldId` — ONE ordered
  key space (disjoint derivation prefixes), values = the
  `SCONE-ENTRY-DOM-V1` (88 fixed bytes) /
  `SCONE-ENTRY-TLD-V1` (variable: 33 bytes private / 1 byte open,
  owner-option enum — flag + conditional owner, no more `until`
  field) encodings;
- `MemoryBackend` (BTreeMap) = reference implementation (dev,
  tests); NO concrete dependency in `scone-blockchain` (redb
  forbidden in the crate) — protocol/consensus depend on the TRAIT
  alone;
- `CachedBackend<B>` (`state_cache.rs`) = bounded LRU cache on top of
  an inner backend: write-through (the inner one is always up to
  date), misses never cached, `range` pages the inner one — same
  answers as the inner backend alone (determinism contract, tested);
- `RedbBackend` (`scone-storage::redb_backend`) = persisted backend:
  `state_kv` table (32-byte raw keys, lexicographic order = required
  ascending order), counter maintained in the `state_kv_meta` meta
  table (never COUNT); `CachedBackend<RedbBackend>` = state with a
  bounded RAM footprint — the "light tier";
- `smt_persist`: consumes the SMT's dirty sets
  (`take_dirty_sets`) and writes them into a backend under
  reserved-prefix keys (0x01 node, 0x02 leaf) — foundation of the
  paged SMT;
- the trait no longer requires `Default` (a disk database has no
  sensible default value);
- `range` iterates in ascending key order (determinism contract) —
  archive path only, never a consensus rule;
- `ChainState` accesses domains/TLDs ONLY by key through the backend
  (point reads, journaled writes).

### Canonical SMT root (pivot)

- `state_root_smt()` (O(1), cached root, `SCONE-STATE-SMT-V1`) is the
  **canonical** state_root: it is what `checkpoint_data` commits and
  what `accept_checkpoint` verifies at the tip. The computation
  NEVER traverses the set of domains;
- the old direct fold `SCONE-STATE-V1` survives as
  `state_root_archive()`: O(N), archive/verification path (full re-fold
  from storage to cross-check a historical root) — not a consensus
  artifact. Both commit the same leaves: equal logical states ⇒ equal
  roots (for both);
- an accepted checkpoint with the SMT root re-verifies after a
  restore (reload): the root is a pure function of the restored state
  (backend + SMT + index).

### Expiry queue (GC O(expired), not O(N))

- `expirations: BTreeMap<u64, BTreeSet<DomainId>>` — expiry instant
  → ids expiring then (ids sorted per slot: state equality and drain
  order independent of arrival order);
- `gc_expired_journaled(now)` drains `range(..=now)` instead of
  scanning the whole state: cost O(expired at that instant);
- RAM bound: one entry per LIVING domain (pending expirations only —
  a renewal re-keys the slot, the GC drains it), never one node per
  domain in a scan structure;
- maintained by the undo journal (bit-exact rollback: the restored
  slot is put back).

### Bounded owner pool (keys, not domains)

- `owner_refcounts: HashMap<OwnerId, u64>` — one counter per DISTINCT
  living owner (domains + TLDs), maintained by the journal on every
  register/transfer/revoke/assign/GC (acquire/release);
- `eligible_validators(now)` reads this index — O(owners), NEVER an
  iteration over domains. Temporal liveness is already guaranteed:
  the deterministic GC (parent timestamp) has removed every expired
  domain AND its refcount before the call. Sorted output
  (deterministic);
- RAM bound: O(distinct owners), independent of the number of
  domains;
- `owner_pool_root` (archive fold) folds these sorted owners — no
  domain scan either.

Steady-state RAM budget (disk/network backend):
O(distinct owners + pending expirations + in-flight grace windows +
undo journal) — all bounded independently of the number of domains,
all restored bit-exact by the journal.

## Seats and the finality committee (exact PoS mechanism)

This section describes the EXACT eligibility and committee-election
mechanism as implemented (`finality.rs`) — the operational
complement (anchor loop, wire) lives in
`/docs/en/technical/relay.md`, the security analysis in
`/docs/en/technical/security-model.md`.

### A seat = possession (no registration)

A PoS **seat** is NOT a registration: it is the possession of a
live domain or TLD. `eligible_validators(now)` computes the pool:

- keys of the live owners via the bounded `owner_refcounts` index
  (one DISTINCT owner per live domain/TLD — a hundred domains
  under one key = ONE seat: the economic anti-Sybil is per
  identity, linear);
- a TLD counts (its claim cost a PoW — the owner of a live TLD is
  a stakeholder), so does a live domain
  (`valid_until > now` — guaranteed by the deterministic GC,
  already drained before the call);
- **minus banned keys** (M9: a slashed equivocator is out of the
  pool for life);
- an **open** TLD (`owner: None`) holds NO seat: its former owner
  is released at opening (`owner_release` in the `SetTldOpen`
  apply), and it dies at GC without ever voting again.

Sorted output (deterministic order, independent of the index's
iteration order).

Structural consequence: **the seat's death = the domain/TLD's
death**. Expiry + GC, `RevokeTld` (seat released),
`TransferTld`/`TransferDomain` (the seat follows the domain to the
new owner), `SetTldOpen` opening (seat released) — every ownership
transition moves or releases the seat in the pool, symmetrically
with the `owner_refcounts` journal (acquire/release).

### Pool FROZEN at the last finalized checkpoint (`FinalizedBase.pool`)

On accepting a checkpoint (`accept_checkpoint`), the finality base
captures the pool of the state AT THE FINALIZED BLOCK:

```text
FinalizedBase { checkpoint, pool, block_ts, seed }
```

- `pool` = `eligible_validators(timestamp of the finalized block)`
  — finality NEVER re-reads the live state afterwards: any
  registration (claim, transfer, ban) after the checkpoint only
  counts for later epochs;
- `seed` = `next_seed(signing_hash(checkpoint))` — the NEXT
  epoch's seed derives from the checkpoint's **content** hash
  (keyed, signature set excluded — two views holding distinct
  quorum subsets of the same checkpoint elect the same
  committee).

### Committee drawing (bootstrap, freshness, degradation)

`committee(recovery)`:

1. **Bootstrap** (no finalized checkpoint): the ENTIRE pool signs
   — a committee drawn from a local view would diverge between
   lagging nodes (the root cause of finality freezes); the whole
   pool is deterministic per the chain. Strict selection resumes
   at the first finality.
2. **Finalized regime**: drawing from the **fresh** subset of the
   frozen pool (RFC proof-of-service §5.2) — each seat (owner
   key) is resolved to its EFFECTIVE operator by the state at the
   tip (`SetTldOperator`/`SetDomainOperator` delegation = a
   recent on-chain fact, or the owner key when self-operated);
   the seat takes part UNDER ITS EFFECTIVE KEY (that is the one
   that signs), provided it has a **fresh heartbeat** (sliding
   window). Banned keys stay excluded.
3. **Safe degradation**: fewer than `MIN_FINALITY_COMMITTEE_SIZE`
   (4) fresh operators → drawing from the ENTIRE pool (cold
   bootstrap, general operator outage — finality is never lost by
   construction).

`allowed_producers(block_ts)` (block production) follows the same
scheme: entire pool at bootstrap; elected committee + recovery
draws (draw k unlocked after k × `epoch_secs` of consensus time)
in the finalized regime.

The checkpoint quorum is `quorum_for(committee size)`
(floor(2n/3)+1); a committee below `MIN_FINALITY_COMMITTEE_SIZE`
never finalizes (BFT guard).

## Deferred to consensus

Deliberately undefined in this crate:

- **mempool**: fine-grained admission/reordering (fees, replacement)
  stays in the relay (M4 brought per-domain cap + windowed TXID
  anti-replay — `REPLAY_WINDOW_BLOCKS = 256`);
- **global ordering**: the mempool sort is a local deterministic sort
  (relay), not a global authority;
- **timestamp authority** beyond GC: expirations (M8b) use the parent
  block's timestamp; other time rules (fine-grained anti-replay)
  remain in consensus;
- distributed storage: the state is in memory; the backend
  (`scone-storage`, redb) applies these rules without duplicating
  them.

Delivered since M8b (previously deferred): per-network registration
PoW (`RegisterTld` always, `RegisterDomain` on an open TLD),
expiry/renewal 1 year + 30-day grace + 3-year horizon, deterministic
GC at the parent block's timestamp, cross-network replay impossible
(genesis + signed field + PoW digest).

## Deterministic distributed simulation (P1.6)

`crates/scone-blockchain/src/sim/` contains a **deterministic**
network simulation harness (`SimNet`): N real `Blockchain`s (no
mocks) driven by a **virtual clock** and an **event queue**
prioritized by `(tick, insertion order)`. No sockets, no async, no
wall clock — every draw (loss, latency, duplication) is a pure
function of `(seed, send number)`, so a scenario with a fixed seed is
reproducible byte for byte.

### Model

- **Transport**: every emission undergoes probabilistic loss
  (configurable in ‰), bounded artificial latency
  (`latency_ticks`), possible duplication (one delayed copy), and the
  current partition map (inter-group messages are lost).
  Duplications never advance the logical clock and trigger no new
  sends: the real event stream is identical with or without
  duplication (determinism contract, proven by the
  `duplicate_and_delay` test).
- **Epidemic gossip**: a node re-transmits to its connected peers
  (same partition) a block/transaction/checkpoint only on first
  sight — relay rule H2 (cuts loops). A parked (orphan) block is
  NEVER re-broadcast; reception gaps are filled by the
  synchronization path (`GetBlocks`), like the relay.
- **Production**: one global production step per interval; each
  partition group produces at most ONE block per round, by its
  authorized node holding the best height. During bootstrap (before
  the first finality) the producer is pinned to the lowest node: the
  authorized producer set derives from state (pool of live domains),
  two slightly different views elect different committees and a
  locally legitimate block would be `InvalidProducer` elsewhere — a
  permanent divergence. After the first finality the committee is
  frozen in the finality base (identical on any converged node) and
  production is free.
- **Finality**: at every multiple of `sign_interval` heights, the
  committee's designated anchor signs the deterministic content of
  its tip via a real crash-safe `SignerGuard` (one key = one vote per
  epoch, state fsynced before diffusion); the other committee members
  sign on receipt (after verifying that the referenced block is indeed
  their canonical block at that height — relay-level guard) and
  re-broadcast the enriched aggregate. At quorum, `accept_checkpoint`
  finalizes. Each node commits to the first proposal seen per epoch
  (bootstrap anti-double-finality guard: two competing proposals of
  the same epoch with disjoint signers cannot both accumulate).
- **Crash/restart**: a crashed node loses all its RAM state (chain,
  mempool, parking, views) but keeps its signer-guard directory on
  disk, exactly like a real process; it resynchronizes by replaying
  the canonical blocks served by its peers
  (`GetBlocks`/`GetCheckpoints`).

### Covered scenarios (`sim::tests`)

| Test | What is proven |
|---|---|
| `convergence_simple` | 8 nodes, 5 ‰ loss, latency 1–5 ticks, 500 blocks: same tip, same `state_root_smt` everywhere; active finality; never two incompatible checkpoints per epoch. |
| `partition_then_heal` | 4a/4b partition for ~300 blocks: both sides produce, healing → convergence with a reorg of the losing branch; crossed checkpoint windows never contradict a shared epoch (before and after healing). |
| `crash_restart` | Crash at h≈200, restart at h≈400: resynchronization by replaying the network's blocks — same height, tip, state; the post-crash transaction is visible to the resurrected node. |
| `duplicate_and_delay` | 10 ‰ duplication: final height, state root, and finalized checkpoint count identical to the run without duplication (idempotence: TxReplay/seen-sets absorb duplicates). |
| `concurrent_tx` | Two concurrent owners on the same domain: exactly one update wins (the owner — `NotOwner` for the intruder, clean typed rejection), never a double state application, the losing tx enters no chain. |

### Parameters and duration bounds

Default values: `produce_interval` 7 ticks, `sign_interval` 16
heights, `sync_interval` 29–60 ticks (41 by default, lowered in some
tests), `horizon` 200k–500k virtual ticks. Every test embeds a
wall-clock guard < 60 s in debug mode
(`assert_under_60s`). The dominant cost is the validation kernel
(~9 ms per empty `push_block_with_gc` in debug, Ed25519
unoptimized): scenarios stay under ~8 nodes × 500 blocks.

### Limits

- **No real transport layer**: this harness exercises chain logic
  under adverse delivery, not the swarm (libp2p, request-response,
  backpressure, discovery). The real e2e tests
  (`crates/scone-network/tests/`) remain the transport proof.
- **In-RAM batch synchronization**: served blocks come from the RAM
  window (scenarios stay under `RAM_WINDOW_BLOCKS`); the
  `BlockPruned` + store-reload path is not simulated (covered by the
  chain's unit tests).
- **Honest anchor model**: anchors only sign content verifiable
  against their own canonical chain; byzantine equivocation
  (deliberate double-signing) is prevented by the `SignerGuard` but
  not exercised as an active attack.
- **One partition at a time**: the group map is global and changed
  sequentially by the driver; continuously overlapping/evolving
  partitions are not modeled.

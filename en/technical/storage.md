# Storage — local node persistence (redb)

> Normative document. Any divergence between this document and
> `crates/scone-storage/src/` is a bug (identify the source of
> truth, fix it). Implementation reference: crate `scone-storage`
> (format v1), `redb` 4.2 backend.

## Purpose

The node's embedded local persistence: canonical blocks, domain
states, DHT cache, metadata. The store is a **persistence**, never
an authority: the RAM state ([`ChainState`], the source of
consensus) is reconstructed at startup from the store, and every
write is a **delta** (domains modified by a block), never a full
copy of the state.

Cross-cutting constraint (user-facing, every milestone): **minimal
memory footprint**. Large volumes (blocks, domain states, DHT
cache) live in redb and are accessed via cursors/ranges — never
loaded fully into RAM, never uselessly duplicated.

## Boundary: the `NodeStore` trait

`crates/scone-storage/src/lib.rs` defines the only storage face
visible to the rest of the code (blockchain, future M4 relay):

- `append_block(height, hash, bytes)` /
  `append_block_with_state(..., deltas)` — atomic append (see
  below);
- `block_at_height(h)` / `block_by_hash(hash)` — point reads;
- `tip() -> (height, hash)` — O(1), read from `meta`;
- `put_domain_state` / `domain_state` / `domain_count` —
  **maintained** counter (`meta["domain_count"]`), never a
  `COUNT(*)`;
- `iterate_domains(after, max) -> DomainPage` — cursor-based
  pagination (batches of 100 by default), ascending byte order of
  the `DomainId`;
- `put_tld_state` / `tld_state` / `tld_count` / `iterate_tlds` —
  same contracts for the TLD registry (`tld_count` maintained,
  bounded cursor) (M7d);
- `put_dht_cache` / `dht_cache` — opaque `SignedDnsRecord` bytes;
- `resolve_name(canonical)` / `resolve_tld(tld)` — persistent
  Name→Id index (P0.2), O(1) point lookup, no iteration over
  domains (default `None` = backend without index, the caller
  falls back to `iterate_domains` or the chain state);
- `tld_floors_all` / `snapshot_tld_floors_all` — read the
  lifetime floors of open TLDs (owner-option: dedicated tables,
  live `tld_floors_v1` and frozen `snapshot_v3_floors`);
- `rewind_to_height(h)` — reorg: removes the blocks above `h` and
  brings `tip`/`tip_height` back to `h` in ONE transaction
  (preliminary to `persist_state_snapshot`, see below);
- `meta_get` / `meta_set`;
- `snapshot_meta` / `snapshot_domains` / `snapshot_tlds` — boot
  state snapshot read (M6b; empty defaults = backend without
  snapshot, `load_chain` then falls back to full loading).

Rules:

- **No redb types in the API**: bytes (`&[u8]`, `[u8; 32]`) and
  core types only. The backend is replaceable.
- **Async-free, no interior mutability**: writes take `&mut self`
  (redb serializes writers), reads take `&self` (one short read
  transaction per call).
- The API never revalidates blocks: the hash passed to
  `append_block*` is the **recomputed** hash supplied by the caller
  (`Blockchain::push_block`).

## redb table schema

| Table | Key (sort) | Value |
|---|---|---|
| `blocks_by_height` | `u64` height (ordered b-tree) | `hash(32) ‖ height(8 BE) ‖ canonical block` |
| `blocks_by_hash` | block hash (32 B) | canonical encoding of the block |
| `domains` | `DomainId` (32 B, byte order) | `DomainStateBytes` (below) |
| `tlds` | `TldId` (32 B, byte order) | `TldStateBytes` (below, M7d) |
| `dht_cache` | `DomainId` (32 B) | `SignedDnsRecord` bytes (opaque) |
| `names_v3` (P0.2) | index key (32 B, byte order) | target id (32 B) |
| `name_reverse_v3` (P0.2) | target id (32 B, byte order) | `tag(1) ‖ canonical name` |
| `meta` | `&[u8]` | `&[u8]` |
| `snapshot_v3_meta` (M6b) | `u64` = 0 (single slot) | `height(8 BE) ‖ tip_hash(32)` = exactly 40 B |
| `snapshot_v3_domains` (M6b) | `DomainId` (32 B, byte order) | `DomainStateBytes` (same format as `domains`) |
| `snapshot_v3_tlds` (M6b) | `TldId` (32 B, byte order) | `TldStateBytes` (same format as `tlds`) |
| `tld_floors_v1` (owner-option) | `TldId` (32 B, byte order) | `u64` (expiry floor, redb implicit BE) |
| `snapshot_v3_floors` (M6b) | `TldId` (32 B, byte order) | `u64` — frozen copy of the floors (state AT the snapshot, not the tip) |

Additional `meta` key (P0.3/P0.4):

| Key | Value |
|---|---|
| `verified_snapshot_v1` (P0.3) | `height(8 BE) ‖ tip_hash(32) ‖ manifest_hash(32)` = 72 B — marker of a VERIFIED imported snapshot (reserved, like `tip`) |

`meta` keys:

| Key | Value |
|---|---|
| `format_version` | `u64 BE` = `STORAGE_FORMAT_VERSION` (1) |
| `tip` | hash (32 B) of the tip block |
| `tip_height` | `u64 BE` (makes `tip()` O(1)) |
| `domain_count` | `u64 BE` (maintained counter) |
| `tld_count` | `u64 BE` (maintained counter, M7d) |

The `hash ‖ height` header of `blocks_by_height` makes each index
self-contained (verifiable without reading the block); both indexes
are updated in the same transaction.

### Genesis convention

The genesis block is deterministic (protocol constants) and is
**never stored** as bytes. A fresh store has `tip == (0,
genesis_hash)`; the first appended block has height 1. The genesis
hash is always re-derived
(`scone_blockchain::genesis_hash()`), never read.

## `DomainStateBytes` encoding format (v2, M8b)

57 or 89 bytes depending on the leading version tag, fixed-width
big-endian integers (canonical, byte-by-byte comparable):

```text
DomainStateBytes = tag ‖ owner(32) ‖ sequence(8 BE) ‖ registered_at(8 BE)
                   ‖ valid_until(8 BE) [‖ record_hash(32)]

tag = 0x01 : record_hash present → exactly 89 bytes
tag = 0x02 : record_hash absent  → exactly 57 bytes
```

- `registered_at` / `valid_until` (M8b): the registration's
  expiration (1 year at claim, renewable, see
  `/docs/en/technical/blockchain.md`).

The M7 encodings (41/73 bytes, no expiration) are **illegitimate**
since M8b: strict decoding → `Corrupted`. (An M7 store regenerates
via replay: `load_chain_replay`.)

- `owner`: `OwnerId` (32 B);
- `sequence`: the last applied `UpdateDomain` sequence (0 right
  after a `RegisterDomain`);
- `record_hash`: the current on-chain commitment of the DNS record
  set (`None` as long as no `UpdateDomain` has been applied).

**Strict** decoding (`DomainStateBytes::decode`): unknown tag,
wrong length, or excess bytes → `StorageError::Corrupted`, never a
panic.

## `TldStateBytes` encoding format (v3, owner-option)

Variable length depending on the enum variant — 34 bytes for a
**private** TLD, 2 for an **open** TLD (neither sequence nor
record hash):

```text
TldStateBytes = tag(0x04) ‖ flag(0x00|0x01) ‖ owner(32)?
```

- `flag` (owner-option, `TldState` enum): `0x00` =
  `TldState::Open` (nobody — self-service + PoW, no owner bytes),
  `0x01` = `TldState::Private` (32 owner bytes follow). Strictly
  canonical — any other byte → `Corrupted`. The invariant "open ⟺
  no owner" is STRUCTURAL: the two states cannot be confused.
- **No more `until` field**: the death deadline of an open TLD is
  not an attribute of the TLD but a fact derived from its domains
  (max of `valid_until + grace`, with a floor set at opening). It
  lives in the dedicated `tld_floors_v1` table (+ its frozen copy
  `snapshot_v3_floors`), rebuilt on replay and restored at boot —
  a restarted node kills the same TLDs as a continuous node.

The M7d (33 bytes, tag `0x01`), M8b (34 bytes, tag `0x02`, boolean
`open`), and V3 (tag `0x03`, embedded `until`) encodings are
**illegitimate**: strict decoding → `Corrupted` (pre-mainnet,
regeneration via replay is the migration path).

The tag exists only for format evolution: a future layout bumps
the tag and old readers fail with `Corrupted` instead of guessing.
**Strict** decoding (`TldStateBytes::decode`): unknown tag, wrong
length, excess bytes, or non-canonical flag →
`StorageError::Corrupted`, never a panic.

## Atomicity guarantees

`append_block_with_state` writes in **a single** redb
`WriteTransaction`:

1. the block in `blocks_by_height` (prior check: the height must
   not exist with different content, otherwise
   `StorageError::Conflict`);
2. the block in `blocks_by_hash`;
3. every domain state delta (`domains`);
4. the `domain_count` counter (incremented by the number of
   **new** domains only; an `UpdateDomain` of an existing domain
   does not increment it);
5. every TLD registry delta (`tlds`, M8b: claims, transfers,
   open/close) — **in the same transaction**: a block written
   without its TLD changes is structurally impossible;
6. the `tld_count` counter (new TLDs only);
7. every **removal** (M8b): domains expired by the deterministic
   GC (`removed_domains`) and revoked TLDs (`removed_tlds`) leave
   the store in the same transaction, counters decremented — a
   restart can never resurrect an expired registration;
8. the **name index** deltas of P0.2 (upserts and removals, see
   below) — same guarantees: a crash never leaves an index entry
   pointing to a missing domain;
9. `tip` and `tip_height`.

redb commits WAL-style: a crash mid-write leaves the **previous**
coherent state. **A block written without its state is therefore
structurally impossible** — this is the milestone's key invariant.

Additional guarantees:

- **Monotonicity**: append at a height ≠ `tip_height + 1` →
  `StorageError::NonMonotonicHeight`, nothing is written (checked
  *before* opening the write transaction);
- **Idempotence**: re-appending the current tip block with the
  same `(height, hash)` pair is a no-op (the fast path compares
  height and hash; at equal height, identical block bytes are also
  a silent no-op, different bytes a `Conflict`);
- **Corrupted database**: truncated/polluted file → typed error
  (`StorageError::Database`/`Corrupted`) at open or on read, never
  a panic (tested);
- **Format version**: a `format_version` ≠ 1 →
  `StorageError::UnsupportedFormat` (an explicit migration path is
  required, never guessing).

## Blockchain integration (`integration.rs`)

Role separation: [`ChainState`] in RAM = consensus authority;
store = persistence. The chosen strategy (the simplest, as
documented): **persist the state at every block** — `store_block`
writes the block + the states of the domains **touched by that
block** (delta). Full replay remains possible and serves as a
control/repair path.

- `store_block(store, chain, block, hash)`: encodes the block
  canonically, reads the final state of the touched domains AND
  claimed TLDs from the RAM state (already updated by
  `push_block`), atomic append.
- `load_chain(store)`: **without replay** — reads tip + tip block
  (O(1) block reads), restores the domain state in pages of 100
  (`DOMAIN_PAGE`) **and the TLD registry in pages of 100**
  (`TLD_PAGE`, M7d — closing the restart window: without it, a
  restarted node forgot every claimed TLD and rejected every
  `RegisterDomain` under that TLD with `UnknownTld`, even though
  its disk held the complete state). The RAM chain keeps only
  genesis + tip; historical blocks stay in the store and are
  served from it. Since M8, `load_chain` then rebuilds the
  **TXID anti-replay index** (see below).
- `load_chain_replay(store)`: replays all stored blocks one by one
  (never all in RAM) with full validation; the resulting tip must
  equal the stored tip, otherwise `Corrupted`.
- `rebuild_replay_index_from_store(store, chain, tip_height)`
  (M8): rescans the last
  `min(REPLAY_WINDOW_BLOCKS, tip_height)` persisted blocks (one
  block in RAM at a time) and reinserts every TxId into the
  chain's index **at its original inclusion height**, via the
  public API `Blockchain::rebuild_replay_index` (no access to
  private fields). Missing block under the tip or undecodable
  within the window → `Corrupted` (those bytes already served to
  build the chain: a failure here signals a damaged store).
- `touched_domains(block)`: ids of the domains modified by the
  block's transactions, deduplicated, block order.
- `touched_tlds(block)`: ids of the TLDs mutated by the block
  (`RegisterTld`, `TransferTld`, `RevokeTld`, `SetTldOpen`),
  deduplicated, block order (M8b).
- `store_block_with_removals(...)`: variant of `store_block` also
  receiving the domains removed by the block's GC (M8b) for the
  atomic write described above. The relay feeds this list from
  `Blockchain::push_block_with_gc` (the application outcome
  exposes `gc_removed_domains`). **GC eviction of the DHT
  cache**: the `dht_cache` entries of removed domains are deleted
  in the SAME transaction as the block (append deltas carry the
  `dht_cache` table) — a DHT record never outlives the domain it
  served, expired entries never accumulate forever (test:
  `gc_evicts_the_dht_cache_entry`).

The relay (M4) will use: `load_chain` at startup, then for every
accepted block `push_block` (RAM) then `store_block` (disk).

## Reorg persistence (`rewind_to_height` + `persist_state_snapshot`)

A reorg (switch to a stronger sister branch) cannot go through the
monotonic append — writing the winning suffix would fail with
`NonMonotonicHeight`. The dedicated path (sticky-minority fix,
relay + store):

1. `rewind_to_height(fork_height)` — ONE redb transaction removes
   the blocks above the fork point and brings `tip`/`tip_height`
   back (the name index and live states are fixed in the next
   step);
2. `persist_state_snapshot(store, chain)` — rewrites the live
   tables (`domains`, `tlds`, name index, TLD floors) from the
   post-reorg RAM state export: the live tables NEVER describe the
   LOSING branch at boot;
3. monotonic re-append of the adopted suffix (ordinary deltas).

A persistence failure on this path is logged and degraded (the
store heals at boot via replay) — never fatal.

## Boot state snapshot (M6b)

`load_chain` as described above is still O(size of the state): no
block is replayed, but the whole `domains` table is re-read. It is
the state-persisted-per-block path that makes replay unnecessary.
The M6b snapshot addresses a different cost: the case where the
RAM state must be reconstructed **by replay** (repair, control, or
a backend without per-block persisted state). It offers a boot in
**O(tip − H)** — replaying only the suffix above the snapshot.

### Write

- **Interval**: `SNAPSHOT_INTERVAL = 64` (public constant of
  `scone-storage`). The trigger is the append of block
  `k·SNAPSHOT_INTERVAL + 1`: that transaction freezes the state of
  the live tables **as it is before applying that block**, i.e.
  the canonical state after block `k·SNAPSHOT_INTERVAL`.
- **Never at the tip**: the snapshot therefore sits at `tip − 1`
  at the closest, always strictly below the tip at the moment it
  is written. A crash right after can never leave a snapshot
  "above" the persisted chain.
- **Atomic**: the freeze (clear + copy of the `domains`/`tlds`
  tables to `snapshot_v3_*`, write of `snapshot_v3_meta`) happens
  in the SAME redb transaction as the append of the trigger block
  — a crash leaves either the old snapshot intact, or the new one
  complete with its block (redb WAL).
- **Single slot**: every snapshot REPLACES the previous one
  (tables emptied then recopied) — no accumulation, no pruning to
  manage, the disk footprint is bounded by ~2× the state.
- The frozen states' format is strictly that of
  `DomainStateBytes`/`TldStateBytes` (reuse, no new encoding);
  only the `snapshot_v3_meta` row is a new format
  (`height(8 BE) ‖ tip_hash(32)`, strict decoding: any other
  length → `Corrupted`).

### Read at boot (`load_chain`)

1. The tip is validated as always (hash recomputed from the tip
   block's header vs `meta["tip"]`).
2. If `snapshot_meta()` exists, its height `H` satisfies
   `0 < H < tip_height`, and the RECOMPUTED hash of the block
   stored at `H` equals the snapshot's `tip_hash` (**anchor** — a
   snapshot foreign to the persisted chain, altered, or from
   another file is detected here), the frozen state is restored in
   pages of 100 then only blocks `H+1..=tip` are replayed (full
   validation, identical to live blocks).
3. The tip reached by the replay must equal the stored tip.
4. **Any failure on this path (missing snapshot, unusable height,
   invalid anchor, undecodable bytes, failed replay) silently
   degrades to the pre-M6b full loading** (reading the live
   tables, without replay) — a bad snapshot never fails the boot.
   Backward compatibility is the particular case "no snapshot": a
   store written before M6b (without the `snapshot_v3_*` tables)
   loads identically, `snapshot_meta()` being `None`.

The replayed suffix is bounded by `SNAPSHOT_INTERVAL` (at most 64
blocks at trigger time + the blocks mined since): the boot is
O(tip − H) with H guaranteed recent.

## TXID anti-replay index rebuilt at boot (M8)

The anti-replay index (`included`, TxId → inclusion height, window
`REPLAY_WINDOW_BLOCKS = 256` — see
`/docs/en/technical/blockchain.md`) is a **pure function of the
canonical chain**: it is never persisted. Since `load_chain` does
not replay history, a restored chain started before M8 with an
**empty** index — a restarted node accepted the re-inclusion of
any transaction still within its replay window (documented
limitation; the state rules remained the only safeguard).

M8 closes this window: after restoration (full path OR snapshot +
suffix replay), `load_chain` calls
`rebuild_replay_index_from_store`, which re-reads the last
`min(REPLAY_WINDOW_BLOCKS, tip_height)` persisted blocks — one
block in RAM at a time, memory-bounded as everywhere — and
reinserts every TxId at its **original height**. A store with
fewer than 256 blocks therefore rebuilds its entire index.

Properties:

- **Decisions identical to the live node**: same entries, same
  pruning rule (`h > tip − 256`) — a node that restarts and a node
  that never stopped reject the same replays (tested: exact index
  count and equality with the pruned live index).
- **Unchanged pruning semantics**: a transaction older than the
  window remains admitted by the index (and then fails the state
  rules if applicable) — exactly as before.
- **The snapshot does NOT freeze the index**: the snapshot's
  suffix replay reconstructs the entries above H, the M8 rescan
  adds the part of the window BELOW H (same contract as
  owner_keys/grace: rebuilt, never persisted).
- **Cost**: ≤ 256 point block reads at boot, O(window) —
  independent of the chain's age.
- A missing or undecodable block **within the window** fails the
  boot with `Corrupted` (those bytes already served to build the
  chain); outside the window, historical blocks are untouched.

`load_chain_replay` (zero trust) rebuilds the index in the same
way through its full replay — both paths converge.

## Persistent Name → Id index (P0.2)

`DomainId = BLAKE3-256(DOMAIN_ID_VERSION ‖ canonical name)` is not
invertible: resolving a name (relay/DNS) requires a persistent
index. P0.2 adds it to the store, with the same transactional
discipline as the rest.

### Schema

Two companion tables, written and erased together:

```text
names_v3        key  = BLAKE3-256("SCONE-NAME-IDX-V1" ‖ canonical name)  (32 B)
                val  = target id (32 B : DomainId or TldId)

name_reverse_v3 key  = target id (32 B)
                val  = tag(0x01 domain | 0x02 TLD) ‖ canonical name
```

- The `names_v3` key is a **hash of the name, not the plaintext
  name** (compact index tables, no name gossip via dumps). The key
  is a **pure function of the name**: the index is rebuildable at
  any time from the chain (`rebuild_name_index`), no additional
  metadata is persisted for it.
- `name_reverse_v3` stores the name (exactly once) because the
  removal paths (GC of expired domains, `RevokeTld`) only know
  **ids**: the hashed key to erase must be found somehow. The tag
  distinguishes the two namespaces.
- Strict separation: `resolve_name` NEVER answers a TLD entry and
  `resolve_tld` never a domain entry (the reverse table's tag
  decides; a `DomainName` has ≥ 2 labels, a TLD has 1 — both key
  derivations share the `names_v3` space but not the results).
- The `DomainId` stays 32 bytes, never reduced (hard project rule:
  consensus identity, collision resistance).

### Transactional consistency

Upserts (`RegisterDomain`, `AssignDomain`, `RegisterTld` — names
carried in plaintext by the transactions) and removals (GC,
revocations — bare ids) are carried by the block's `StateDelta`
(`name_upserts` / `name_removals`) and applied in the **same redb
transaction** as the block and state writes. A crash can therefore
leave neither an index entry pointing to a missing domain (no
ghosts), nor a registered domain without its entry. A failed
append writes nothing (tested). The named variants
`put_domain_state_with_name` / `put_tld_state_with_name` /
`remove_domain_state` offer the same atomicity to repair tools.

`store_block` / `store_block_with_removals` derive the name deltas
from the SAME transactions of the block — no extra parameter at
the relay interface.

### Rebuild (`RedbStore::rebuild_name_index`)

The index is a pure function of the canonical chain, like the M8
anti-replay index. `rebuild_name_index` rebuilds it from the
**persisted blocks** (the `DomainStateBytes` do NOT contain the
name — owner/sequence/expiration only — so the source of truth is
the transaction stream, where `RegisterDomain` and `AssignDomain`
carry the canonical name in plaintext):

1. pass 1: scan `blocks_by_height` genesis→tip, one block in RAM
   at a time; collect the `name → id` bindings seen in
   `RegisterDomain`/`AssignDomain`/`RegisterTld`;
2. pass 2: filter to **living** targets only (the persisted state
   is authoritative) — a domain registered then GC'd/expired is
   not reindexed;
3. pass 3: erase + full rewrite of both tables in ONE transaction
   (crash → old or new complete index).

A missing or undecodable block → `Corrupted` (same bytes that
built the chain). The cost is O(distinct names seen) in RAM (≤ 253
B per name) and O(blocks) in reads — a repair tool, not a boot
path.

### What the index is NOT

- Not an authority: the RAM chain state remains the only
  authority; the index is a rebuildable cache (a divergence is a
  transactionality bug, detectable by `rebuild_name_index`
  compared against it).
- Not frozen in the M6b snapshot: the `names_v3`/`name_reverse_v3`
  tables are live and follow each block's deltas; the state
  snapshot does not embed them (reconstruction by deltas on
  resume, or `rebuild_name_index` as repair).

### What the snapshot does NOT commit

The RAM state `ChainState` carries internal structures not
committed by consensus (owner keys, grace windows). They are NOT
frozen: the suffix replay rebuilds them by applying the
transactions (deterministic, same content as the full replay for
the relevant window). The TXID anti-replay index follows the same
contract (M8, see previous section): rebuilt at boot, never
frozen. Bit-exact equality with `load_chain_replay` is verified by
test on the canonical state (state_root V2 + SMT).

## VERIFIED state snapshot — chunks + manifest + crypto verification (P0.3)

The M6b snapshot above speeds up the boot of a node that trusts
its disk. P0.3 addresses a different need: transporting a state
between nodes (bootstrapping a new node, repair) without trusting
the transport. The format and verification live in
`crates/scone-storage/src/verified_snapshot.rs` + the root
reconstruction in
`crates/scone-blockchain/src/snapshot_verify.rs`.

### Trust model

The snapshot carries the state of a **finalized** block — a
checkpoint signed by the committee quorum (`CheckpointData.state_root`,
the O(1) `state_root_smt` root). The checkpoint's authenticity
(quorum of Ed25519 signatures) is validated by the chain layer
BEFORE import. The import itself verifies snapshot ↔ checkpoint
adherence by RECOMPUTATION:

1. **hash of every page**: `blake3("SCONE-SNAP-PAGE-HASH" ‖
   full page encoding)` must equal the hash announced by the
   manifest (one falsified byte → rejection);
2. **sum of entries == manifest counters** (a missing page or a
   wrong counter → rejection);
3. **SMT root recomputed from the imported entries ==
   `checkpoint.data.state_root`** — THE cryptographic check: the
   entire state (domains + TLDs) is re-folded into a fresh SMT
   (`snapshot_verify::recompute_state_root`, leaves
   `SCONE-LEAF-DOM-V2`/`SCONE-LEAF-TLD-V4` exactly like the live
   path) and the root must equal the one the committee signed. A
   falsified, truncated, or incomplete snapshot is rejected
   BEFORE ANY WRITE;
4. **structural consistencies**: `height`/`tip_hash`/`state_root`
   of the manifest == those of the checkpoint; strictly
   increasing ids (no duplicates); sequential page indexes; DoS
   bounds (≤ 10,000 entries/page, ≤ 2^20 pages, total budget).

Verification failure → `StorageError::SnapshotRejected`, the store
remains strictly unchanged (everything is verified before the
write transaction).

### Serialized format (strict, big-endian, merciless decoding)

```text
manifest = "SCONE-SNAP-MAN-V1"
        ‖ height(8) ‖ tip_hash(32) ‖ state_root(32)
        ‖ page_count(4) ‖ domain_count(8) ‖ tld_count(8)
        ‖ page_hashes(page_count × 32)

page    = "SCONE-SNAP-PAGE-V1"
        ‖ index(4) ‖ height(8) ‖ state_root(32)      ← every page is
        ‖ entry_count(4) ‖ entries                   self-identifying

entry   = kind(0x01 domain | 0x02 TLD | 0x03 TLD floor) ‖ id(32)
          ‖ [len(2) ‖ state | floor(8 BE)]
          (state = DomainStateBytes 57/89 B or TldStateBytes 2/34 B —
           the existing canonical storage formats, reused; kind
           0x03 (lifetime floor of an open TLD, owner-option) has
           a dedicated format WITHOUT a length prefix:
           id(32) ‖ floor(8 BE), fixed size)
```

Page order: all domain entries (increasing ids), then all TLD
entries (increasing ids), then the TLD lifetime floors FROZEN at the
snapshot height (`snapshot_tld_floors_all` — not the live table) —
the order of the store's cursor
pagination. Every page carries `(index, height, checkpoint's
state_root, entries)`: a lone page knows which snapshot it belongs
to. Pages of 256 entries at export
(`VERIFIED_SNAPSHOT_PAGE_ENTRIES`); DoS bounds at import. TLD
floors (`tld_floors_v1`) are part of the portable format: kind
0x03 carries them, and the import requires them strictly
ascending and targeting a TLD PRESENT in the snapshot —
otherwise the importer would never kill what an exporter would
kill (silent fork, rejected).

### Export (`export_verified_snapshot(store, checkpoint)`)

Serializes the state of the LAST FINALIZED CHECKPOINT — not the
tip. Emits domains, TLDs, then the TLD lifetime floors frozen at the
snapshot height (export/import symmetry: the export historically
omitted floors — fixed, regression test
`export_carries_tld_lifetime_floors`). Prerequisite: the store holds an M6b persisted snapshot
(`snapshot_v3_*`) exactly at the checkpoint's height (the
`CHECKPOINT_KEEP`/`SNAPSHOT_INTERVAL` window guarantees such a
state is regularly available); the block's anchor is RECOMPUTED
from the stored bytes and must equal
`checkpoint.data.block_hash`. Otherwise →
`StorageError::SnapshotUnavailable` (export is an explicit
action: wait for the next interval/checkpoint).

### Import (`import_verified_snapshot(store, checkpoint, manifest, pages)`)

Verifies everything (see above), then writes in ONE redb
transaction: live tables `domains`/`tlds` + counters + `tip = (H,
checkpoint hash)`, frozen tables `snapshot_v3_*` anchored at the
same block, and the reserved marker
`meta["verified_snapshot_v1"]` (`height ‖ tip_hash ‖
manifest_hash`). Target: empty store only (otherwise
`SnapshotUnavailable`).

### Verification cost (documented)

SMT reconstruction via ordered insertions: N entries × O(40)
hashes ≈ O(N·40). ~2.1 M blake3 hashes for 100,000 domains (tens
of ms). ONE-OFF — a bootstrap/repair import, never a per-block
path: acceptable. Ordered insertion (increasing ids) creates each
branch point only once; in any case the root depends only on the
content (SMT guarantee, differential fuzz in `smt.rs`).

## State bootstrap — without genesis replay (P0.4)

A new node joins the network without replaying history from
genesis:

```text
finalized checkpoint (relay, validated by the chain layer)
        │
        ▼
verified snapshot (manifest + pages, crypto verification above)
        │  import → state at H + tip (H, hash) + marker
        ▼
anchor block H (requested from the network BY HASH — append_bootstrap_anchor)
        │
        ▼
recent blocks H+1..H+k (relay, increasing height, ordinary atomic
        │               append — the storage API requires only this)
        ▼
synced — boot replays ONLY the suffix above H
```

- **Storage API exposed to the relay**: `append_bootstrap_anchor(H,
  hash, bytes)` (lays the anchor block: required marker, height/
  hash consistency with the marker, idempotent) then ordinary
  atomic append by increasing height. Network retrieval (by hash
  from the manifest) is a relay concern, not a store one.
- **`load_chain`** recognizes the marker: history below H is
  legitimately absent (non-archive node). Concretely: the M8
  anti-replay window rescan is floored at H (blocks below H do
  not exist and are not missing); a tip without a stored block
  AFTER an import (anchor not yet laid) → typed diagnostic
  `SnapshotUnavailable` ("the anchor must come from the network"),
  not corruption.
- **Archive vs full-state node**: an archive node (operator flag)
  keeps everything — it serves historical blocks to the network
  and boots via the existing paths. A full-state node (P0.4
  default for a new node CONFIGURED this way) boots at H: it
  validates future blocks (suffix replay) but cannot serve
  history below H or replay the genesis. **The default behavior
  does NOT change**: this path is only active if the operator
  explicitly triggers the import (`import_verified_snapshot` is a
  dedicated action, never automatic); every existing node boots
  identically.
- **Equality with the full node**: after joining the suffix
  H+1..H+k, the state, tip, and `state_root_smt` are bit-exact
  with a full node at the same tip (tested); the anti-replay
  index covers exactly the txs at/above H — a non-archive node
  cannot know the window below H, the state rules remain the
  safeguard (a duplicated tx below H fails on state, never
  accepted twice).

## Memory policy

- **Cursors everywhere**: `iterate_domains` bounds the page to
  `max` entries (100 by default) via redb's lazy `range`; the
  entire domain table is never materialized. Large lists (TLDs,
  domains) mandatorily go through this cursor (`iterate_tlds`,
  same contract, M7d).
- **Delta-only**: `append_block_with_state` writes only the
  domains and TLDs modified by the block — never the full state;
- **No synchronous duplication**: the RAM state is not
  re-persisted per block; the disk state is not reloaded per
  block;
- **Point reads**: every read opens a short read transaction,
  returns the value, closes; redb `AccessGuard`s live only for
  the duration of the bounded copy;
- **Historical blocks never in RAM**: `load_chain` loads only the
  tip; `load_chain_replay` streams height by height;
- **Maintained counter**: `domain_count` is O(1), not a scan.

## DoS bounds

The store bounds what a hostile peer can make it persist or make
it allocate:

- **`MAX_DHT_CACHE_ENTRY` (64 KiB)**: a larger DHT record is
  rejected before any write transaction by `put_dht_cache` →
  `StorageError::TooLarge` (nothing is written);
- **`MAX_DOMAIN_PAGE` (10,000)**: `iterate_domains` *internally*
  bounds the requested page to `min(max, 10_000)` — an excessive
  `max` never materializes more than 10,000 states in RAM;
  `iterate_tlds` has the same bound (`MAX_TLD_PAGE`, M7d);
- **Reserved `meta` keys**: `meta_set` rejects `tip`,
  `tip_height`, `format_version`, `domain_count`, `tld_count`,
  and `verified_snapshot_v1` (P0.3) with
  `StorageError::ReservedKey` — overwriting the store's internal
  accounting would silently corrupt it.
- **Double-bounded DHT cache (LRU)**: `DHT_CACHE_MAX_ENTRIES`
  (100,000 entries) AND `DHT_CACHE_MAX_BYTES` (2 GiB — hostile
  case guard; each entry is already capped at 64 KiB by
  `MAX_DHT_CACHE_ENTRY`), whichever bound hits first evicts by
  key order. The counters (entries + bytes) are maintained at
  write time — never a scan to count; the exact rebuild happens
  only at open (the single full scan, at boot). Accessors
  `dht_cache_entries`/`dht_cache_bytes`, tunable bounds via
  `set_dht_cache_bounds` (test:
  `dht_cache_respects_the_double_bound`).

## Trust model

Two loading paths, two levels of guarantee:

- **`load_chain` — local disk trust**: the tip's hash is
  **recomputed** from the tip block's header and compared to the
  stored `meta["tip"]` (divergence → `StorageError::Corrupted`,
  never a chain restored on a forged hash), and the domain states
  are strictly decoded. However, historical blocks are **not
  replayed**: the persisted states are taken at face value. This
  is the fast startup of a node that trusts its own disk (written
  only by that same node, atomically). Since M6b, this path first
  uses the **state snapshot** when usable (see below): the
  snapshot's anchor is recomputed and the suffix above H is
  revalidated like live blocks — the replayed portion is thus
  zero-trust, the frozen base disk-trust.
- **`load_chain_replay` — zero trust**: all blocks are re-read
  and revalidated one by one (full validation, identical to live
  blocks); the resulting tip must equal the stored tip. This is
  the control/repair tool.

**Recommendation**: a strict node (or a node whose storage file
could have been tampered with — shared disk, restore of a dubious
backup) must start via `load_chain_replay`, not `load_chain`.

## Concurrency

redb 4.x allows **a single `Database` object per file and per
process** (simultaneous double `open` fails with
`DatabaseAlreadyOpen`). The supported model: a cloned `RedbStore`
(`Arc<Database>`) shared across threads — writers serialized by
`&mut self` + redb's internal lock, concurrent readers (snapshot
read transactions). After all handles are fully dropped, the file
can be reopened.

## Tests (`tests/redb_store.rs` + `#[cfg(test)]` modules)

Every test uses its own redb tmpfile (`tempfile`). Coverage:

- block roundtrip genesis→N, restart, re-read, replay;
- M7d: TLD registry persisted by atomic delta, restored on
  restart (a `RegisterDomain` under a TLD claimed before the
  restart remains admissible — window closed), exact counters on
  reopen, ascending `tlds` cursor, failed append leaves no TLD,
  reserved key `tld_count` rejected;
- atomicity: failed append leaves nothing (tip, domains, index
  intact); duplicate deltas counted once;
- domain cursors: pagination 100/7, ascending `DomainId` order,
  start strictly after an existing or missing id, `max = 0`;
- exact `domain_count` counter (insert/update/reopen);
- DHT cache roundtrip + overwrite;
- concurrency: reads from two threads on cloned handles;
- corrupted database: truncated / garbage file → typed error, no
  panic (`catch_unwind`);
- M6b: snapshot written at the interval, strictly below the tip;
  boot from snapshot = full bit-exact replay (state_root V2 +
  SMT, complete logical state, tip); altered snapshot (corrupted
  tip_hash, undecodable state, height > tip, wrong-length meta)
  → silent fallback to full loading, same final chain; replayed
  suffix < interval; snapshot replacement (no accumulation) when
  crossing the interval; pre-M6b store without snapshot tables
  loads identically;
- M8: anti-replay index rebuilt at boot — a tx re-included within
  the window is rejected (`TxReplay`) after restore; a tx older
  than the window is admitted (pruning unchanged, state rules
  decide); exact count == live index (==
  `load_chain_replay`) for a store both shorter and longer than
  the window; snapshot boot: the part of the window below H is
  rebuilt by the rescan, the part above by the suffix replay;
- P0.2: name index — `resolve_name == DomainId::from_name` for
  120 domains (live, after restart, after wipe + rebuild),
  `resolve_tld` for the claimed TLD; GC of an expired domain
  removes the entry (no ghosts), before and after rebuild; strict
  separation of the domain/TLD namespaces; a failed append leaves
  no index entry; the index key is the documented BLAKE3
  derivation;
- P0.3/P0.4 (`tests/verified_snapshot.rs`): export → import →
  boot == full replay (state, tip, bit-exact `state_root_smt`;
  the imported state is that of the finalized block, not the
  tip); page falsified by one byte → rejection before any write
  (store strictly unchanged); missing page → rejection; wrong
  `state_root` (manifest ↔ checkpoint, then recomputed root ≠
  signed root) → rejection; wrong counters / heights /
  inconsistent tips / duplicate ids → rejection; import requires
  an empty store; export requires a persisted snapshot at the
  checkpoint's height; anchor contract (`append_bootstrap_anchor`:
  required marker, consistency, idempotence); strict serialized
  format (excess bytes, tags, DoS bounds); snapshot at H then
  blocks H+1..H+k joined → chain identical to the full node
  (state/tip/root), boot without history below H, anti-replay
  index = exactly the txs at/above H.

[`ChainState`]: ../../../crates/scone-blockchain/src/state.rs

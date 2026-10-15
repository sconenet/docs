# RFC — Proof of operation and finality committee (proof-of-service)

> Status: NORMATIVE — implemented end to end (steps 1 through 5:
> core/protocol/blockchain/network/CLI; this document is the
> consolidated form). Every rule below is verified against the code;
> the constants cited are the coded values.
> Date: October 2026 (normative consolidation). Touches:
> `scone-core`, `scone-protocol`, `scone-blockchain`,
> `scone-network`, `scone` (CLI).

## 1. Motivation

The finality pool is the owner pool (owners of live domains/TLDs).
Owning grants a seat — **operating** grants nothing. This RFC ties
the right to sign checkpoints to proof of operation of an online
relay, without ever conditioning ownership:

- **Preserved freedom**: a private TLD remains assignable with no
  activity condition for as long as its term is renewed
  (`RenewTld`, 3-year term). Only *checkpoint signing* is earned
  through presence.
- **Incentive to operate**: operating a private TLD (or any seat)
  = sitting on the committee. The operator takes part in keeping
  the network up and the protocol recognizes it.
- **Measurable decentralization**: the committee becomes the set of
  operators *actually online*, not dormant keys.

## 2. Overview

A lightweight `Heartbeat` transaction (discriminant `0x71`),
signed by an **operator** key, emitted once per epoch. The state
holds a sliding window of recent heartbeats. An epoch's committee
is elected **from the subset of the pool whose effective operator
has a fresh heartbeat**. Without a heartbeat: no signing, but no
loss of ownership rights.

## 3. The Heartbeat transaction

```
Heartbeat {
    operator: PublicKey,     // Ed25519 key of the operator (the signer)
    owner: OwnerId,          // identity RE-DERIVED from operator (never trusted)
    tip_seen: [u8; 32],      // last tip seen by the node
    height_seen: u64,        // corresponding height
    timestamp: u64,          // Unix seconds
} + Ed25519 signature over the canonical payload (network injected)
```

Signed body (wire, `scone-protocol`):
`operator[32] ‖ owner[32] ‖ tip_seen[32] ‖ height_seen.v ‖
timestamp.v ‖ operator[32]` (the signer's key appears at the tail
of the body, as for every tx) `‖ signature[64]`, i.e. ~230 bytes
encoded — **no PoW**: the cost is continuous presence (one
heartbeat per epoch, for months).

The signer IS the operator: `owner` is always re-derived from
`operator` (never taken from the input), and decoding rejects with
a typed error (`InvalidOwner`) a signature key that differs from
the head `operator` field.

- `tip_seen`/`height_seen` serve observability (detection of buggy
  or lagging operators); they do NOT influence validity (a lightly
  lagging node is still a live node).

### 3.1 Pure validation (scone-core)

1. Valid Ed25519 signature over the network's canonical payload.
2. `owner` == identity derived from `operator` (`InvalidOwner`
   otherwise) — checked at encode, at decode AND by `validate()`.

### 3.2 State rules (application, scone-blockchain)

Upon application of a heartbeat in a carrying block of height
`block_height` (every rejection precedes any mutation):

1. **Cross-epoch anti-replay**: `timestamp` strictly increasing
   per operator — typed rejection `StaleHeartbeat` if
   `timestamp <= operator_last_hb[operator]`.
2. **Per-epoch uniqueness**: only one heartbeat accepted per
   (`operator`, epoch of the carrying block) — duplicate rejected
   with a typed error (`HeartbeatDuplicate`). The epoch is
   `block_height / epoch_min_blocks` (same definition as finality,
   never a new constant).
3. Journaled insertion into the window + update of
   `operator_last_hb` (reversible in a reorg, bit-exact undo).

### 3.3 Anti-spam

- 1 accepted tx per operator per epoch — the maximum flow is
  `pool_size / epoch_secs`, negligible.
- No fees (pre-economy): the rate is structurally bounded.

## 4. Operation delegation

The seat → operator link goes through two dedicated transactions:

- `SetTldOperator { tld_id, operator }` (discriminant `0xA9`),
  signed by the **current** owner of the **private** TLD (an open
  TLD has no one to sign legitimately — typed rejection
  `TldIsOpen`). `operator: None` = revocation, back to
  self-operation by the owner's key; instantly revocable.
- `SetDomainOperator { domain_id, operator }` (discriminant
  `0x5E`), signed by the current owner of the domain. Same
  semantics.
- `RegisterTld` / `RegisterDomain` / `AssignDomain` carry NO
  `operator` field: the seat → operator link is always a recent
  on-chain fact, resolved at the tip.

The operator option on the wire is canonical: `0x00` = `None`
(self-operation), `0x01` = raw 32-byte key; any other value is
rejected at decode.

The state maintains:

```
seat_operators: HashMap<OwnerId, OperatorKey>   // per seat
operator_seats: HashMap<OperatorKey, u64>       // bounded counter
operator_last_hb: HashMap<OperatorKey, u64>     // last accepted ts
```

Application rules (`apply_seat_operator`, journaled):

- **Idempotence**: re-delegating to the same operator (or
  revoking a non-existent delegation) is a non-journaled no-op
  (bit-exact undo requires it).
- **Anti-rental bound**: an operator can only be delegated by
  **one single seat** at a time (`MAX_SEATS_PER_OPERATOR = 1`). A
  second seat delegating to an already-taken operator → typed
  rejection `OperatorAlreadyDelegated`. Rationale: without a
  bound, a large operator sells signing in bulk and recreates
  centralization — exactly what the elected committee is meant to
  avoid.

## 5. Freshness window

### 5.1 Sliding window

```
recent_heartbeats: BTreeMap<(epoch, OperatorKey), ()>
```

- Entry inserted upon application of a valid heartbeat (§3.2).
- Deterministic purge at EVERY block application
  (`gc_heartbeats_journaled`): any entry of epoch `e` with
  `e + HEARTBEAT_WINDOW_EPOCHS <= epoch` is removed (journaled —
  reversible in a reorg, like the expiry queues).
- `HEARTBEAT_WINDOW_EPOCHS = 2`: the current epoch + the previous
  one — an operator may miss ONE epoch without losing freshness,
  the second one expires it.
- **Structural memory cap**: the window holds at most
  `HEARTBEAT_WINDOW_EPOCHS × operator_count` entries (one per
  operator per epoch, tuple key); `operator_last_hb` is bounded by
  the number of operators and is never purged (the anti-replay
  rule is monotone).

### 5.2 Freshness

"Fresh" = presence of an entry `(e, operator)` for `e` in
`[epoch − HEARTBEAT_WINDOW_EPOCHS + 1, epoch]`. Since the state
purges at every block, presence in `recent_heartbeats` IS
freshness (`operator_is_fresh`).

### 5.3 Effective operator

A seat's **effective operator** is `seat_operators[seat]` if a
live delegation exists, otherwise the owner's key
(`effective_operator`). The seat signs under that effective key —
a delegation moves the signing right, never the ownership.

## 6. Committee election

```
eligible = { effective operators of the frozen pool's seats
             (banned key excluded) having a fresh heartbeat }

if |eligible| >= MIN_FINALITY_COMMITTEE_SIZE (= 4) :
    committee = select_committee(seed, eligible)   // existing draw
else :
    committee = select_committee(seed, pool)       // safe degradation
```

Normative details (verified in `finality.rs`, specified in
`/docs/en/technical/blockchain.md` § "Seats and the finality
committee"):

- the `pool` is the pool **frozen at the last finalized
  checkpoint** (`FinalizedBase.pool` — never the live state): a
  local view would diverge between nodes and the quorum would
  never form;
- the `seed` derives from the previous checkpoint's content hash
  (`signing_hash`);
- every seat of the frozen pool is resolved to its EFFECTIVE
  operator by the state at the tip (delegation and heartbeats are
  recent on-chain facts: two nodes at the same tip compute the
  same set); banned keys stay excluded;
- the draw runs over the sorted/deduplicated set of effective
  keys (`select_committee` unchanged).

**Explicit and safe degradation**: a bootstrapping network (no
heartbeat), or a general outage of operators, falls back to the
entire pool — finality is never lost by construction. This is the
project's "never deadlock" rule applied here.

### 6.1 Cold bootstrap

Epoch 0 (and as long as fewer than `MIN_FINALITY_COMMITTEE_SIZE`
operators are fresh): the entire pool. The constraint naturally
tightens as operators emit. No special transitional period to
configure.

## 7. Automatic emission (scone-network)

At every production cycle (`Wake::Produce` — no new timer), the
relay checks whether an EPOCH BOUNDARY was crossed
(`height / epoch_min_blocks`, same definition as finality). At the
first cycle of a new epoch, it emits a heartbeat if:

1. an anchor key is armed (`--anchor-key` or auto), AND
2. that key itself holds a live seat (LOCAL check: the key is in
   the on-chain eligibility pool — owners of live domains/TLDs,
   banned keys excluded).

Local anti-spam: at most ONE emission per epoch per node
(`last_emitted_epoch` memo, local memory, never chain state); the
memo is only taken after a successful mempool injection — a
failure retries at the next cycle. Nothing here is fatal: at worst
the heartbeat waits for the next cycle.

A **delegated operator** (who is not themselves an owner) is not
covered by automatic emission: they emit via the CLI builder of
§8 (manual or air-gapped operation). The heartbeat is NOT
necessarily produced into a block by the emitter itself: it is a
tx like any other (mempool, production by anyone).

## 8. CLI (`scone`)

The offline heartbeat builder (removed with the `scone tx` surface) was
removed with the offline tx surface (cli-simplify-v1). Relays
emit their own heartbeat automatically (§7); manual/air-gapped
emission, if ever needed again, would come back as a dedicated
command. Defaults were: `tip_seen` = 32 zero bytes
(observability-only), `height_seen` = 0, `timestamp` = now.
- As for every tx, `tx sign` **rebinds** operator and owner to the
  REAL signing key (input values never trusted); `tx verify`
  validates locally.

## 9. Observability

- `NodeStatus` (wire p2p message `0x0d`) carries
  `fresh_operators: u32` (varint): the number of operators with a
  fresh heartbeat — the live subset of the pool eligible for the
  committee (effective operators of the pool's seats, banned keys
  excluded, sorted/deduplicated). Filled by the relay from
  `state().fresh_operators(height)`; displayed by `scone status`
  as `fresh operators: N`.
- Campaign argument: "operating a private TLD = sitting on the
  finality committee of the decentralized DNS" — verifiable
  on-chain.
- Future surface (unspecified): `operator_status` — per seat, the
  delegated operator, freshness (fresh / expired / never seen),
  last `tip_seen`. The local control RPC was removed (p2p-only
  node); the surface is still to be defined.

## 10. What this RFC does NOT do

- **No loss of possession** for inactivity (option A — decay of
  the un-operated TLD — is explicitly rejected).
- **No reward** (option C awaits an economic layer).
- The heartbeat does not prove the *quality* of service (blocks
  produced, DHT records served) — only presence. Counting real
  contribution is a possible future extension, more gameable.

## 11. Protected invariants

- Bit-exact determinism: the window and the election are pure
  functions of the applied state (journaled, bit-exact undo).
- Never panic on network data: malformed heartbeat or delegation
  → typed rejections (`InvalidOwner`, `StaleHeartbeat`,
  `HeartbeatDuplicate`, `OperatorAlreadyDelegated`, `TldIsOpen`,
  `NotTldOwner`, `NotOwner`).
- Possession is unconditional; only checkpoint signing is
  conditional.

## 12. Constants and parameters (coded values)

| Name | Value | Location |
|---|---|---|
| `HEARTBEAT_WINDOW_EPOCHS` | 2 | `scone-blockchain/src/state.rs` |
| `MAX_SEATS_PER_OPERATOR` | 1 | `scone-blockchain/src/state.rs` |
| `MIN_FINALITY_COMMITTEE_SIZE` | 4 | `scone-core/src/consensus_params.rs` |
| `Heartbeat` discriminant | `0x71` | `scone-protocol/src/transaction.rs` |
| `SetTldOperator` discriminant | `0xA9` | same |
| `SetDomainOperator` discriminant | `0x5E` | same |

These values are part of the consensus: changing them breaks chain
replayability (window/purge) or the election (committee).

## 13. Implementation plan — final state

| Step | Crate | Content | State |
|---|---|---|---|
| 1 | core + protocol | `Heartbeat`, `SetTldOperator`, `SetDomainOperator`, wire, validation | done |
| 2 | blockchain | sliding window (journaled), filtered pool, election, ~15 tests | done |
| 3 | network | automatic emission, tx relay | done |
| 4 | CLI | `tx build heartbeat`, `fresh operators` display in `scone status` | done |
| 5 | docs | this RFC consolidated into normative form, fr/en mirror | done |

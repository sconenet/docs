# RFC — Proof of operation and finality committee (proof-of-service)

> Status: IMPLEMENTED (steps 1-3: core/protocol/blockchain/network;
> automatic emission from scone-network is in place). Step 4 is
> partial: `tx sign`/`tx verify` accept a heartbeat, but neither
> `tx build heartbeat` nor the fresh-operator count in `scone status`
> exist. Step 5 (RFC → consolidated normative form) remains to be
> done.
> Date: October 2026. Touches: `scone-core`, `scone-protocol`,
> `scone-blockchain`, `scone-network`.

## 1. Motivation

The current finality pool is the owner pool (owners of live
domains/TLDs). Owning grants a seat — **operating** grants nothing.
This RFC ties the right to sign checkpoints to proof of operation of
an online relay, without ever conditioning ownership:

- **Preserved freedom**: a private TLD remains immortal and
  assignable with no activity condition. Only *checkpoint signing*
  is earned through presence.
- **Incentive to operate**: operating a private TLD (or any seat)
  = sitting on the committee. The operator takes part in keeping the
  network up and the protocol recognizes it.
- **Measurable decentralization**: the committee becomes the set of
  operators *actually online*, not dormant keys.

## 2. Overview

A new lightweight transaction `Heartbeat`, signed by an
**operator** key, emitted once per epoch. The state holds a sliding
window of recent heartbeats. An epoch's committee is elected **from
the subset of the pool whose operator has a fresh heartbeat**.
Without a heartbeat: no signing, but no loss of ownership rights.

## 3. The Heartbeat transaction

```
Heartbeat {
    operator: PublicKey,     // Ed25519 key of the operator relay
    tip_seen: BlockHash,     // last tip seen by the node
    height_seen: u64,        // corresponding height
    timestamp: u64,          // Unix seconds
} + Ed25519 signature over the canonical payload (network injected)
```

Properties:
- ~120 bytes encoded; **no PoW** — the cost is continuous presence
  (one heartbeat per epoch, for months).
- `tip_seen` serves observability (detection of buggy or lagging
  operators); it does NOT influence validity (a lightly lagging node
  is still a live node).

### 3.1 Validation (acceptance rules)

1. Valid Ed25519 signature over the network's canonical payload, and
   `owner` == identity derived from `operator` (`InvalidOwner`
   otherwise).
2. `timestamp` strictly increasing per operator across epochs
   (anti-replay of an old signed heartbeat — typed rejection
   `StaleHeartbeat` at application time).
3. **Only one heartbeat accepted per (`operator`, epoch)** — a
   duplicate within the same epoch is rejected with a typed error
   (`HeartbeatDuplicate`). The epoch is that of the carrying block
   (`height / epoch_min_blocks`, same definition as finality).

### 3.2 Anti-spam

- 1 tx per operator per epoch — the maximum flow is
   `pool_size / epoch_secs`, negligible.
- No fees (pre-economy): the rate is structurally bounded.

## 4. Operation delegation

The seat → operator link:

- `SetTldOperator { tld_id, operator }` (signed by the private TLD's
  owner; an open TLD has no one to sign legitimately) —
  `operator: None` = revocation, back to self-operation by the
  owner's key; instantly revocable.
- For domains: `SetDomainOperator { domain_id, operator }` (signed
  by the domain owner). Same semantics.
- `RegisterTld` / `RegisterDomain` / `AssignDomain` carry NO
  `operator` field: delegation goes through the two dedicated
  transactions above (the seat → operator link is always a recent
  on-chain fact, resolved at the tip).

The state maintains:

```
seat_operators: HashMap<OwnerId, OperatorKey>   // per seat
operator_seats: HashMap<OperatorKey, usize>     // bounded counter
```

**Anti-rental bound**: an operator can only be delegated by **one
single seat** at a time (`MAX_SEATS_PER_OPERATOR = 1`). A second
seat delegating to an already-taken operator → typed rejection
`OperatorAlreadyDelegated`. Rationale: without a bound, a large
operator sells signing in bulk and recreates centralization —
exactly what the elected committee is meant to avoid.

## 5. Freshness window and election

### 5.1 Window

```
recent_heartbeats: BTreeMap<(epoch, OperatorKey), ()>
```

- Entry inserted upon application of a valid heartbeat.
- Deterministic purge at every block application: entries that left
  the `HEARTBEAT_WINDOW_EPOCHS = 2` epoch window are removed
  (journaled — reversible in a reorg, like the expiry queues). With
  2: the current epoch + the previous one — an operator may miss ONE
  epoch without losing freshness, the second expires it.
- "Fresh" = presence of an entry for the operator in the current
  window.

### 5.2 Committee election (modification of `select_committee`)

```
eligible(pool) = { seats of the pool whose associated operator
                   (or the owner's key if self-operated)
                   has a fresh heartbeat }

if |eligible| >= MIN_FINALITY_COMMITTEE_SIZE :
    committee = select_committee(seed, eligible)   // existing draw
else :
    committee = select_committee(seed, pool)       // safe degradation
```

Implementation details (verified in `finality.rs`, specified in
`/docs/en/technical/blockchain.md` § "Seats and the finality
committee"): the `pool` is the pool **frozen at the last finalized
checkpoint** (`FinalizedBase.pool` — never the live state), the
`seed` derives from the previous checkpoint's content hash
(`signing_hash`), each seat is resolved to its EFFECTIVE operator
by the state at the tip (delegation = a recent on-chain fact) and
takes part under its effective key; banned keys stay excluded.

The **degradation** is explicit and safe: a bootstrapping network
(no heartbeat), or a general outage of operators, falls back to the
entire pool — finality is never lost by construction. This is the
project's "never deadlock" rule applied here.

### 5.3 Cold bootstrap

Epoch 0 (and as long as fewer than `MIN_FINALITY_COMMITTEE_SIZE`
operators are fresh): the entire pool. The constraint naturally
tightens as operators emit. No special transitional period to
configure.

## 6. Automatic emission (scone-network)

The relay emits one heartbeat per epoch when:

1. an anchor key is armed (`--anchor-key` or auto), AND
2. its identity (or its delegate) holds a live seat.

Emission happens on the epoch's first production cycle — no new
timer, it rides with `Wake::Produce`. The heartbeat is NOT
necessarily produced into a block by the emitter itself: it is a tx
like any other (mempool, production by anyone).

## 7. Observability

- Planned, not yet exposed: `operator_status` — per seat, the
  delegated operator, freshness (fresh / expired / never seen),
  last `tip_seen` (the local control RPC was removed; the surface
  is to be defined on the p2p side).
- `scone status` does not yet display the number of fresh
  operators / the pool size (the data exists in the state:
  `fresh_operators`).
- Campaign argument: "operating a private TLD = sitting on the
  finality committee of the decentralized DNS" — verifiable
  on-chain.

## 8. What this RFC does NOT do

- **No loss of possession** for inactivity (option A — decay of the
  un-operated TLD — is explicitly rejected).
- **No reward** (option C awaits an economic layer).
- The heartbeat does not prove the *quality* of service (blocks
  produced, DHT records served) — only presence. Counting real
  contribution is a possible future extension, more gameable.

## 9. Protected invariants

- Bit-exact determinism: the window and the election are pure
  functions of the applied state (journaled, bit-exact undo).
- Never panic on network data: malformed heartbeat → typed
  rejection (`InvalidOwner`, `StaleHeartbeat`, `HeartbeatDuplicate`,
  `OperatorAlreadyDelegated`).
- Possession is unconditional; only checkpoint signing is
  conditional.

## 10. Implementation plan

| Step | Crate | Content | Size | State |
|---|---|---|---|---|
| 1 | core + protocol | `Heartbeat`, `SetTldOperator`, `SetDomainOperator`, wire, validation | M | done |
| 2 | blockchain | sliding window (journaled), filtered pool, election, ~15 tests | L | done |
| 3 | network | automatic emission, tx relay | S | done |
| 4 | CLI | `tx build heartbeat` (debug), status display | S | partial (`tx sign`/`verify` only) |
| 5 | docs | this RFC → normative, protocol.md/finality update | S | to do |

## 11. Open decisions (to be settled by Benoît)

1. `HEARTBEAT_WINDOW_EPOCHS = 2` — comfortable or too loose?
   (2 epochs = an operator can miss one epoch without losing its
   signing seat.) *This is the value coded today.*
2. `MAX_SEATS_PER_OPERATOR = 1` — or tolerate 2-3 for small
   operators hosting several namespaces? *Coded value: 1.*
3. Expose `operator_status` via RPC at step 3 or later?
   *Became moot: the local control RPC was removed (p2p-only
   node) — the surface is still to be defined.*

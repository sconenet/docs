# RFC — Proof of operation and finality committee (proof-of-service)

> Status: IMPLEMENTED (steps 1-2: core/protocol/blockchain; automatic
> emission from scone-network follows its own workstream).
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

1. Valid Ed25519 signature over the network's canonical payload.
2. `timestamp` within a reasonable window of the carrying block
   (± `epoch_secs` — historical anti-replay bound).
3. **Only one heartbeat accepted per (`operator`, epoch)** — a
   duplicate within the same epoch is rejected with a typed error
   (`HeartbeatDuplicate`). The mempool applies the same exclusion
   while pending (replaces the old non-included one).
4. `timestamp` strictly increasing per operator across epochs
   (anti-replay of an old signed heartbeat).

### 3.2 Anti-spam

- 1 tx per operator per epoch — the maximum flow is
   `pool_size / epoch_secs`, negligible.
- No fees (pre-economy): the rate is structurally bounded.

## 4. Operation delegation

The seat → operator link:

- `RegisterTld` / `RegisterDomain` / `AssignDomain` gain an optional
  field `operator: Option<PublicKey>`. Default: `None` =
  self-operation by the owner key.
- New transaction `SetTldOperator { tld, operator }` (signed by the
  TLD owner) — instantly revocable.
- For domains: `SetDomainOperator { domain, operator }` (signed by
  the domain owner). Same semantics.

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
- Deterministic purge at every epoch boundary: entries older than
  `HEARTBEAT_WINDOW_EPOCHS = 2` epochs are removed (journaled —
  reversible in a reorg, like the expiry queues).
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

- RPC `operator_status`: per seat, the delegated operator, freshness
  (fresh / expired / never seen), last `tip_seen`.
- `scone status` displays the number of fresh operators / pool size.
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
  rejection (`InvalidHeartbeat`, `HeartbeatDuplicate`,
  `OperatorAlreadyDelegated`).
- Possession is unconditional; only checkpoint signing is
  conditional.

## 10. Implementation plan

| Step | Crate | Content | Size |
|---|---|---|---|
| 1 | core + protocol | `Heartbeat`, `SetTldOperator`, `SetDomainOperator`, wire, validation | M |
| 2 | blockchain | sliding window (journaled), filtered pool, election, ~15 tests | L |
| 3 | network | automatic emission, tx relay | S |
| 4 | CLI | `tx build heartbeat` (debug), status display | S |
| 5 | docs | this RFC → normative, protocol.md/finality update | S |

## 11. Open decisions (to be settled by Benoît)

1. `HEARTBEAT_WINDOW_EPOCHS = 2` — comfortable or too loose?
   (2 epochs = an operator can miss one epoch without losing its
   signing seat.)
2. `MAX_SEATS_PER_OPERATOR = 1` — or tolerate 2-3 for small
   operators hosting several namespaces?
3. Expose `operator_status` via RPC at step 3 or later?

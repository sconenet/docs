# Network relay (M4) — architecture, flows, and CLI control

> Normative specification of the Scone network relay (`scone-network`).
> Any doc/code divergence = bug (fix the source of truth).

## Overview

The **relay** is Scone's full P2P node: it maintains the canonical
blockchain (RAM + `NodeStore` redb), relays transactions and blocks,
produces blocks (devnet mode), serves the sync protocol, publishes and
resolves `SignedDnsRecord`s in a Kademlia DHT, and exposes the
**p2p protocol** (`/scone/reqres/1`) that the CLI queries directly
(RPC→p2p unification: the JSON TCP channel is gone).

```text
                    ┌────────────────────────────────────────┐
                    │              scone relay               │
  P2P (libp2p)      │                                        │
  QUIC (udp,        │  Swarm libp2p                          │
  ephemeral port    │  ├─ identify  (address discovery)      │
  by default) ◄─────┤  ├─ ping      (liveness)               │
                    │  ├─ kad       (record DHT)             │
                    │  └─ reqres    (scone-prot. messages)   │
                    │                                        │
                    │  Relay core (single task)              │
                    │  ├─ Blockchain (full validation)       │
                    │  ├─ Mempool (bounded, dedup by TxId)   │
                    │  ├─ BlockBuilder (devnet production)   │
                    │  ├─ Fork parking (bounded, 2048)       │
                    │  └─ Sync driver (batched catch-up)     │
                    │                                        │
  CLI ◄─────────────┤  reqres p2p (QUIC, same messages)      │
                    └────────────────┬───────────────────────┘
                                     │
                              NodeStore (redb)
```

`crates/scone-network` is the **only async crate** in the workspace:
everything below it (crypto → core → protocol → blockchain →
storage) remains synchronous and pure.

## Connection admission (P1.5)

Hard limits at the swarm level (libp2p `connection_limits` behavior,
the first entry of `SconeBehaviour`):

- `MAX_TOTAL_CONNECTIONS = 128` — cap on established connections
  (incoming + outgoing) per relay; excess incoming is REFUSED,
  never queued;
- `MAX_PER_PEER_CONNECTIONS = 4` — a single peer cannot monopolize
  the total quota by opening many connections;
- `MAX_PENDING_INCOMING = 32` — slowloris guard: unauthenticated
  half-connections never wait indefinitely.

These limits complement the existing per-message limits
(`MAX_MESSAGE_LEN` 1 MiB) and per-budget limits (sync `max ≤ 128`
blocks, DNS TCP 64 connections/512 B/5 s). A connection-flood
attacker gets refusals, not memory growth.

## Peer discovery

Three complementary mechanisms, in this priority order:

1. **Explicit seeds** (`--bootstrap`, full multiaddrs) — always take
   priority; providing a non-empty list REPLACES everything else.
   `--no-builtin-seeds` empties the embedded seeds for 100% private
   deployments.
2. **Embedded seeds** (`builtin_seeds(network)` in
   `scone-network/src/config.rs`) — frozen at each official binary
   release, per network. An unconfigured relay joins the network
   alone. (Empty on testnet as long as the public testnet is not
   deployed.)
3. **Learned directory** (`<data-dir>/peers.json`) — every address
   learned via `identify` is retained (last seen; cap of **2000
   entries** — addresses, not connections, ~400 KiB RAM at the
   maximum — and a **90-day age cutoff**: an entry last seen more
   than 90 days ago is dropped at write time and ignored at load
   time, keeping the 2000 slots filled with plausible peers instead
   of pre-churn fossils whose only contribution would be a
   boot-time dial timeout) and saved ~1×/min at the production
   tick. On boot with no seed and no bootstrap, the directory is
   recomposed: the relay finds its usual peers without
   configuration. It is a cache: missing/corrupted file = empty
   directory, never an error.
4. **LAN mDNS** (`--mdns`) — zero-config discovery on the
   local link: neighbouring relays on the same network segment
   find each other with NO seed at all (the zero-config devnet
   path). **Default: ON for devnet, OFF elsewhere** (a resettable
   local network is a discovery network by design; a public
   deployment multicasts nothing). Behaviour: bind of the multicast socket; a bind failure
   (no multicast on the interface, sandboxed environment)
   **disables** mDNS with a warning — never fatal (discovery is a
   convenience, never a lifeline). Every `Discovered` neighbour
   feeds the Kademlia table (`kad.add_address`), exactly like an
   explicit bootstrap minus the configuration; `Expired` entries
   are tracked by libp2p itself (the connection closes on its own
   timeout). The behaviour's `mdns`/`upnp` members are `Toggle`s
   (enabled/disabled at construction,
   `build_behaviour(key, enable_mdns, enable_upnp)`).
5. **UPnP** (`--upnp`, opt-in) — QUIC listen-port mapping by the
   home router: the node becomes reachable from the outside
   WITHOUT a circuit relay. On `NewExternalAddr`, the external
   address is registered as the swarm's real external address
   (`add_external_address` — seen by identify/autonat, a
   transport-relay candidate). No IGD gateway (`GatewayNotFound`)
   or non-routable gateway (`NonRoutableGateway`): a plain info
   log, the mapping is skipped — normal cases on a home LAN. An
   expired mapping (failed renewal) is a warning.

mDNS follows the network (an explicit `--mdns`/`SCONE_MDNS`
always wins): **ON by default on devnet**, OFF on
testnet/mainnet; UPnP stays **disabled by default**
(`enable_mdns`/`enable_upnp` = `false` in `config.rs`, the relay
turns mDNS on for devnet): a public deployment multicasts
nothing and talks to no router.

### Local topology optimization (always on)

The relay actively optimizes its neighborhood using the pure
`scone-topology` engine (see the crate's docs). **Always enabled**;
the only exposed tuning is a **profile** (`--topology-profile`,
env `SCONE_TOPOLOGY_PROFILE`, default `default`):

| Profile | min / target / max | eval. | for whom |
|---|---|---|---|
| `tiny` | 2 / 4 / 6 | 120 s | Raspberry Pi, home box, 1-vCPU VPS |
| `small` | 4 / 8 / 12 | 90 s | small server, home NAS |
| `default` | 8 / 16 / 32 | 60 s | standard server (default) |
| `high` | 16 / 32 / 64 | 45 s | sovereign node, backbone |
| `very-high` | 32 / 64 / 128 | 30 s | founder hubs |
| `devtest` | 2 / 4 / 6 | 5 s | **LAB/TEST only** (unstable by design) |

```
scone relay --topology-profile tiny
```

The protective parameters (hysteresis 0.15, cooldown 600 s, grace
120 s, measurement 15 s) are identical everywhere: they guard
stability and have no reason to scale with the node size. An
unknown name is a typed startup error.

An inconsistent config (`min > target > max`, threshold outside
`[0,1]` — `NaN` included —, zero interval) is a **typed startup
error**, never silent behavior.

**View (`TopologyView`)** — fed by swarm events:

- successful `ping` → RTT observation (median window of 8 samples);
- failed `ping` → reliability penalty;
- `identify` received → addresses (≤ 4/peer) + announced protocols
  (kad/reqres/identify/relay bitmask);
- connection established → connected state, direction
  (incoming/outgoing), age, detected circuit-relay;
- connection closed → penalty + churn counter ONLY if unsolicited
  (a voluntary eviction does not count);
- outgoing dial failure → reliability penalty.

The view is bounded (500 peers; beyond that, only connected peers
are memorized).

**`kad_required` from the kbuckets** — before each evaluation, the
relay walks the Kademlia routing table: a peer is protected if it is
`Connected` in a bucket, or `Disconnected` in a SATURATED bucket
(k = 20). A protected peer is NEVER chosen for eviction.

**Optimization loop** — at the profile's evaluation interval
(60 s under `default`; see the table above):

1. recompute `kad_required` (kbuckets);
2. engine evaluation (hysteresis: 120 s grace period, minimum score
   improvement of +15%, eviction cooldown 600 s);
3. decision applied: `AddCandidate` → dial the candidate;
   `Replace` → close the evicted peer (ONCE — progressive, no
   looping) then dial the replacement;
4. a **metrics** snapshot is logged (INFO level):
   known/connected/incoming/outgoing/circuit/protected counts,
   mean/median RTT, distinct IP buckets, max concentration, churn
   (replacements/disconnections).

The neighborhood score is NEVER a trust measure (see the
`scone-topology` docs): no consensus decision derives from it.

### Reachability (AutoNAT) and transport relay (circuit-relay v2)

Every relay runs AutoNAT (client + server): the node asks its peers
to attempt a dial BACK to it; the verdict (`reachable`: yes =
reachable from the outside, no = behind NAT, unknown = not yet
tested) is refreshed every 5 min and exposed in `NodeStatus` (field
`nat_reachable`, wire varint 0/1/2) — the CLI displays it
(`scone status`). Private addresses count
(`only_global_ips = false`): a LAN mesh tests its LAN reachability.

A reachable relay enables circuit-relay v2 **hop** (it can relay
traffic for NATed peers) and **DCUtR** (hole-punching: two nodes
connected via circuit attempt a direct connection and release the
relay). A relay deemed NATed automatically requests a reservation
from the first known reachable peer and listens through it (address
`/p2p/<relay>/p2p-circuit`).

Production rule: a relayed NATed node should not become a block
producer (degraded latency/availability) — the PoS rule (seat =
living domain) does not forbid it yet; to be documented as an
operational requirement before the public testnet.

## Transport and libp2p protocols

| Protocol | Name | Role |
|---|---|---|
| QUIC (quic-v1) | — | Main transport (udp) |
| identify | `/scone/id/1` | Peer address discovery (fills the Kademlia table) |
| ping | libp2p standard | Liveness |
| Kademlia | `/scone/kad/1` | DHT: key = `DomainId` (32 B), value = canonical encoding of the `SignedDnsRecord` |
| request-response | `/scone/reqres/1` | scone-protocol application messages |

### Application frame (`/scone/reqres/1`)

Each frame is `len: u32 BE || bytes` where the bytes are the
canonical encoding of a `Message` from `scone-protocol` (the
existing codec — no second format). The length is bounded by
`limits::MAX_MESSAGE_LEN` **before** any read or allocation: a peer
announcing a larger frame is rejected at no memory cost.

Exchanged messages (one per request/response round-trip):

| Message | Direction | Reply |
|---|---|---|
| `Hello { version }` | handshake | `Hello` (local version) |
| `Ping` / `Pong` | liveness | `Pong` |
| `GetBlock { hash }` | query | `Block` (RAM window) or `Pong` if unknown |
| `GetBlocks { start, max }` | sync | **one** `Block` (height `start`), or `Pong` if none |
| `Block (b)` | block relay | `Pong` (ack) |
| `Transaction (tx)` | tx relay | `Pong` (ack) |
| `GetRecord { domain_id }` | local DHT | `Record` (local cache) or `Pong` |
| `Record (r)` | publication | `Pong` if accepted, error otherwise |

Rule: **exactly one reply per request**; acknowledgment messages
are `Pong`. Sync replies carry one block at a time — catch-up
iterates on the requester side (see "Sync"). This is deliberately
bounded: never more than one block in flight per peer.

## Relay behavior

### Startup

0. network resolution (`--network testnet|mainnet|devnet`, testnet default
   while the project is in dev — M8b). The data-dir is **per
   network** (`~/.scone/<network>/`): a non-empty store that holds a
   chain from another network is refused (`Corrupted`);
1. open/create the redb store (`<data-dir>/chain.redb`);
2. `load_chain(store, network)`: O(1) tip + paginated domain states
   (RAM window: genesis + tip);
3. swarm construction (ephemeral per-session Ed25519 identity —
   persisting the network identity is an open decision);
4. compose the boot list (see
   [Peer discovery](#peer-discovery)): explicit `--bootstrap`
   (must end with `/p2p/<peer-id>`) ELSE the network's embedded
   seeds ELSE the learned directory (`peers.json`);
5. dial each multiaddr and register it in the Kademlia table
   (a dead dial is ignored — the others make up the network).

### Receiving a transaction (P2P or CLI `submit tx` submission)

1. strict bounded decoding (`MAX_MESSAGE_LEN`);
2. full cryptographic validation (`validate_transaction`:
   owner/key binding, `verify_strict` over the canonical payload);
3. **anti-replay FIRST** (cheap, state-independent, definitive),
   then state pre-check (cheap mirror of the `push_block` rules —
   PoW, which is costly, is only verified at push time): a
   replayed `RenewDomain` answers `TxReplay`, not a lying PoW
   error (its proof binds the expiry the first application moved
   past). Network
   separation is enforced at step 2: the signature payload injects
   the RELAY's network, a tx signed elsewhere fails with
   `InvalidSignature` (wire 2026-09: no more `network` field);
   `RegisterDomain` → TLD of the registered name
   (`UnknownTld` otherwise, D1/M7c) **and open** (`TldClosed`
   otherwise, M8b) then name free; `UpdateDomain` → known domain,
   exact sequence `current + 1`, owner; `RegisterTld` → free TLD;
   `TransferTld`/`RevokeTld`/`SetTldOpen` → known TLD + owner
   signer; `AssignDomain` → TLD owner + free name;
   `RenewTld` → known private TLD + owner, fixed term re-derived, grace-window precheck; `RenewDomain` → known domain + owner, renewal PoW precheck
   (bound to the current expiry) at the next height;
4. mempool insertion (fixed capacity, dedup by `TxId` —
   re-submitting a tx already in the pool is a no-op);
5. broadcast to peers (except the sender).

### Receiving a block (P2P, sync, or self-produced)

1. `push_block`: existing full validation (parent = canonical tip,
   height, recomputed Merkle root, every transaction validated then
   applied — atomic per block);
2. if accepted: the relay reads the application outcome
   (`push_block_with_gc` → hash + domains removed by the
   deterministic GC) and persists via `store_block_with_removals`
   (atomic delta append: block + tip + modified states — upserts
   AND removals: expired domains, revoked TLDs). A restart can
   therefore never resurrect an expired registration;
3. mempool reconciliation (included txs are removed);
4. broadcast to peers (except the sender).

An invalid block is rejected with a typed error; the relay keeps
running. No panic on network data. Heavy applies (Argon2 + redb
commit) go through `tokio::task::block_in_place` (sync driver,
p2p accept, production): the runtime moves ready tasks to other
workers — applying a block no longer stalls the DNS service or
the metrics.

### DNS snapshot (publication)

After EVERY applied block (p2p accept — reorgs included —, sync
driver, production) AND at boot (a restart must not NXDOMAIN
live names until the next block), the relay publishes the
**authoritative DNS snapshot** onto the `watch` feed read by the
DNS surface (full specification in `dns.md`):

- tip extension (the overwhelming path): **O(touched) delta**,
  copy-on-write — the block's transactions name the touched
  domains, plus the deterministic GC removals
  (`AppliedBlock.gc_removed_domains`, evaluated at push time);
- reorg / boot: **full rebuild** (one paged
  `ChainState::dns_facts` walk).

### Block production (M5: signed producer)

Every `produce_interval` seconds (2 s by default): if the mempool
is non-empty, a block is assembled by `BlockBuilder`
(`timestamp = now`) **signed by an authorized producer** (payload
`SCONE-BLOCK-V1` in `header.consensus`, see
`/docs/en/technical/blockchain.md`) and follows exactly the
acceptance path of a received block. A relay with no txs produces
nothing.

Production key: the anchor key (`--anchor-key`) when the relay is
armed with one — an anchor is an authorized producer — otherwise
the fixed devnet key, valid as long as the producer pool is empty
(open bootstrap: genesis). As soon as a living domain/TLD exists, a
block signed by the devnet key is rejected (`InvalidProducer`).

### Anchor loop (finality, M5)

With `--anchor-key` (`.sconekey` keyfile, passphrase via
`--anchor-passphrase-env`, default `SCONE_ANCHOR_PASS` — the
`scone-network` library default `SCONE_ANCHOR_PASSPHRASE` is never
reached through the binary, clap always overrides it):

- after each accepted/produced block, if this node is a member of
  the current committee **and** the committee reaches
  `MIN_FINALITY_COMMITTEE_SIZE`, the relay signs the proposed
  checkpoint via its `SignerGuard` (`signer-state.dat` in the
  data-dir — one key = one vote/epoch, crash-safe fail-closed) and
  gossips it;
- received signatures (`Message::Checkpoint`) are accumulated by
  `CheckpointData` hash (bounded to 256 entries); at
  `quorum_for(committee size)` member signatures on the SAME
  data → `accept_checkpoint` + broadcast of the finalized
  checkpoint;
- a relay without an anchor key stays **passive**: it accumulates,
  finalizes, and relays, but never signs;
- `GetCheckpoints` → the `checkpoint_window()` window is served;
- any loop error is logged and swallowed — never fatal;
- the status (`GetStatus`) exposes `checkpoints` and `finalized_epoch`.

### Fork parking and reorgs (sticky-minority fix)

A sister block at the same height, or a block whose parent is not
the current tip (`ParentNotTip`/`UnknownParent`), is NEVER lost:
it is **parked** (bounded map, `PARKED_CAP = 2048` blocks, ~KiB
each — bounded worst-case RAM; a full parking silently ignores
newcomers). Hashes already offered to `accept_block` are
deduplicated (`seen_blocks`): a re-gossiped losing block is not
re-evaluated.

At EVERY `Produce` tick, `reconsider_parked`:

1. linear drain of parked blocks whose parent became the tip;
2. if a parked branch is stronger than the current tip (fork-choice
   rules UNCHANGED: finality floor, longest chain, lowest hash),
   adoption — the displaced suffix is itself parked (BOTH sides
   of the fork stay evaluable).

This un-sticks a node that adopted the losing side of an
equal-height fork: the winning block, offered once then refused,
is periodically re-evaluated instead of being forgotten forever
(finding chaos-30: two sister blocks 15 ms apart).

**Reorg persistence**: adopting another branch cannot go through
the monotonic append (`NonMonotonicHeight`). The relay applies
`rewind_to_height(fork point)` then `persist_state_snapshot`
(rewrite of the live state from the chain's export) then a
monotonic re-append of the winning suffix (see `storage.md`). A
persistence failure is logged and degraded (heals at boot via
replay) — never fatal.

## Sync (catch-up)

On each connection to a peer, the relay requests
`GetBlocks(start = local_height + 1)`. Each received `Block` that
exactly completes the expected height triggers the request for the
next height, until exhaustion (`Pong` = nothing more) or
`MAX_SYNC_ROUNDS` (10,000) batches — an absolute bound even against
a lying peer. Received blocks go through full validation: a
malicious peer cannot get anything invalid accepted.

### Record DHT

- **put** (API/web flow — the `record put` CLI was removed): strict bounded decoding
  (`MAX_DHT_CACHE_ENTRY`), verification against on-chain state
  (owner, sequence, recomputed `record_hash` — see "Record
  verification"), then `kad.put_record` (local storage +
  replication to the closest peers, `Quorum::Majority` of the
  replication factor) and persistent redb cache.
- **get** (`record get`): `kad.get_record`; the waiter is bound
  to its `QueryId` and the requested `DomainId` — a reply only
  resolves the matching query, and the record key must be exactly
  the requested domain (checked **before** on-chain verification).
  The result is then verified against the chain before being
  returned. Cap on simultaneous requests: `MAX_DHT_WAITERS` (256);
  beyond that, immediate error. A successfully verified resolution
  is written to the persistent cache (M5): local reads
  (`domain_info`) then serve it without a new DHT query.
- Kademlia runs in server mode, with no record TTL (the chain
  governs validity, not time).

### Record verification ("the chain is the authority" rule)

A resolved or received `SignedDnsRecord` is only accepted/returned
if:

1. the domain is registered on-chain;
2. `record.owner` = current on-chain owner;
3. `record.sequence` = current on-chain sequence;
4. `record_hash(record.record)` (recomputed BLAKE3) = the
   `record_hash` committed on-chain.

Otherwise: silent rejection on the P2P side, explicit error on the
CLI side. The record's Ed25519 signature is **verified on receipt**
by the relay (`ResolveLocal`: the owner's public key is known
on-chain, the signature must unlock — a record signed by another
key is rejected, `record rejected`); it is also transported for
future verifiers — the on-chain commitment remains the primary
integrity control.

## Prometheus metrics

An HTTP endpoint (Prometheus text format 0.0.4, hand-rolled
exposition — no new dependency) served by a dedicated task:

- **Public by default** on `127.0.0.1:9101` (the CLI applies that
  default; the library default is `None` — several relays in one
  process never collide).
- `--metrics-addr ADDR|off`: another address, or `off` = endpoint
  disabled.
- `--metrics-token TOKEN`: switches to **private** mode —
  `Authorization: Bearer <TOKEN>` required (constant-time
  comparison, `401` otherwise; the token never appears in logs
  nor in the exposition).
- A bind failure **disables the endpoint** (non-fatal warning) —
  never the node.

The effective bind is announced on stdout (machine-readable
contract, like `p2p:`): a `metrics: <addr>` line as soon as it
listens.

**19 counters** (`counter`, monotonic — hot paths:
`fetch_add(Relaxed)` on a shared `Arc<Metrics>`, never a
consensus input):

| Group | Series |
|---|---|
| Chain | `scone_blocks_applied_total`, `scone_blocks_rejected_total`, `scone_checkpoints_signed_total`, `scone_checkpoint_sigs_received_total` |
| Mempool | `scone_txs_admitted_total`, `scone_txs_rejected_total`, `scone_txs_evicted_grace_total`, `scone_txs_included_total` |
| DNS | `scone_dns_queries_total`, `scone_dns_cache_hits_total`, `scone_dns_cache_misses_total`, `scone_dns_local_chain_total`, `scone_dns_dht_answers_total`, `scone_dns_rejected_unverified_total` |
| DHT | `scone_dht_pulls_issued_total`, `scone_dht_negative_hits_total`, `scone_dht_negative_expired_total`, `scone_dht_negative_capped_total`, `scone_dht_cache_evictions_total` |
| Peer network | `scone_peer_connections_established_total`, `scone_peer_connections_closed_total`, `scone_peer_connections_incoming_total`, `scone_peer_dial_failures_total`, `scone_peer_blocks_received_total`, `scone_peer_transactions_received_total` |

**6 gauges** (`scone_height`, `scone_mempool`, `scone_peers`,
`scone_domains`, `scone_finalized_epoch`, `scone_addr_book_entries`)
— the first five refreshed every 15 s via the `RelayCommand` channel
(same path as the CLI `status` — single source of truth);
`scone_addr_book_entries` is maintained directly by the registry
(updated at every address-book insertsource of truth).

E2E test: `metrics_endpoint_public_and_private` (200 +
exposition, 401 without/with a wrong token, 200 with Bearer).

## CLI control (RPC→p2p unification)

The JSON TCP RPC server has been **removed**: the CLI (and the
registrar's axum bridge) speak the p2p protocol **directly** —
same wire (`/scone/reqres/1`), same handlers on the relay side,
zero parallel channel. Every command opens a QUIC connection to
the node's multiaddr (`--peer`), exchanges ONE `Message`, closes.

Control messages served by the relay:

- `GetStatus` → `NodeStatus { network, nat_reachable, tip,
  height, domain_count, mempool, peers, finalized_epoch,
  checkpoints, fresh_operators }` (displayed by `scone status` —
  `fresh_operators` = operators with a fresh heartbeat, RFC
  proof-of-service);
- `LookupName { name }` → `NameState` (on-chain state, like
  `lookup`);
- `GetOwnedNames { owner }` → `OwnedNames` (live domains + TLDs
  of an `OwnerId`);
- `Transaction (tx)` → submission with free gossip (the relay
  rebroadcasts to peers);
- `GetRecord { domain_id }` / `Record (r)` — DHT resolution and
  publication (like `scone record get` on the CLI side);
- application rejection → `Message::Error(msg)` (reaches the
  client as a readable message, not an EOF).

The client brings up a minimal libp2p swarm (request-response +
identify), 30 s call timeout (20 s round trip). Default `--peer`:
`/ip4/127.0.0.1/udp/4001/quic-v1/p2p/12D3KooW…` (local dev node;
the full multiaddr with the real peer id is announced by the
relay on stdout via the `p2p:` line).

The internal `RelayCommand` channel survives for the embedded DNS
surface (`ResolveLocal`, one in-process round trip, never a DHT
waiter — see `dns.md`).

## CLI

```
scone relay [--network testnet|mainnet|devnet] [--data-dir DIR] [--listen MULTIADDR] [--bootstrap MULTIADDR]... [--no-builtin-seeds]
            [--mdns] [--upnp] [--rpc ADDR (obsolete, ignored)]
            [--anchor-key PATH] [--no-anchor-auto] [--anchor-passphrase-env VAR] [--genesis FILE]
            [--geo-config FILE] [--geo-ip-table PATH] [--geo-country U16] [--geo-continent U16] [--geo-asn U32] [--geo-region U32]
            [--health-check] [--no-health-check] [--health-ttl-secs U64] [--health-port U16]
            [--topology-profile tiny|small|default|high|very-high|devtest]
            [--dns ADDR] [--dns-upstream ADDR]... [--dns-cache-limit U] [--dns-cache-max-mem SIZE] [--dns-max-inflight U]
            [--metrics-addr ADDR|off] [--metrics-token TOKEN]
scone status [--peer MULTIADDR]
scone domains [--identity ID | --pubkey HEX] [--peer MULTIADDR]
scone domain register NAME [--identity ID] [--dir DIR] [--passphrase-env VAR] [--peer MULTIADDR]
scone domain set NAME TYPE VALUE [TTL] [--identity ID] [--peer MULTIADDR]   # express single record
scone domain update NAME --file FILE [--identity ID] [--peer MULTIADDR]
scone domain get NAME [--peer MULTIADDR]
scone domain renew NAME [--identity ID] [--peer MULTIADDR]
scone record get NAME [--peer MULTIADDR]

# --peer defaults to the local relay (SCONE_PEER env overrides);
# --identity defaults to the single keystore identity when there
# is exactly one. Removed (stubs suggest replacements): show,
# submit, lookup, owned, record put.
```

Every control command speaks QUIC directly to the relay's
multiaddr (`--peer`, default local node — RPC→p2p unification, no
more JSON TCP channel). Relay absent → clear message "p2p node
unreachable — is a relay running?", non-zero exit. The `--rpc`
flag is kept but **obsolete and ignored** (the RPC server was
removed; script compatibility only).

### Environment configuration (`.env`)

At startup, `scone` loads (via `dotenvy`, before clap parsing) a
`.env` file from the current directory: missing file = silent
no-op, never an error. The loaded values become the
**environmental baseline** of the options that declare a variable
(`env = "…"` in `cli.rs`); **an explicit CLI flag always wins**
over `.env`, and `.env` wins over the built-in defaults:

```text
explicit CLI flag  >  .env variable  >  built-in default
```

The `.env.example` file at the repository root documents every
variable (copy to `.env` and adjust; `.env` is **gitignored** —
never commit a real `.env`). Variables:

| Variable | Mirror flag | Semantics |
|---|---|---|
| `SCONE_NETWORK` | `--network` | `testnet` (default), `mainnet`, `devnet` |
| `SCONE_DATA_DIR` | `--data-dir` | data directory (default `~/.scone/<network>/`) |
| `SCONE_LISTEN` | `--listen` | P2P listen multiaddr (default: all interfaces, ephemeral port) |
| `SCONE_BOOTSTRAP` | `--bootstrap` | bootstrap seeds, **repeatable** (one line per value — dotenvy accumulates); replaces the builtin seeds |
| `SCONE_GENESIS` | `--genesis` | genesis file (founder nodes) |
| `SCONE_MDNS` | `--mdns` | LAN mDNS discovery (`true` = enabled, `false` = off; default: ON on devnet, OFF elsewhere) |
| `SCONE_UPNP` | `--upnp` | UPnP port mapping (`true` = enabled) |
| `SCONE_DNS_ADDR` | `--dns` | UDP DNS server bind address |
| `SCONE_DNS_UPSTREAM` | `--dns-upstream` | recursive fallbacks, **repeatable** |
| `SCONE_DNS_CACHE_LIMIT` | `--dns-cache-limit` | DNS cache entry cap |
| `SCONE_DNS_CACHE_MAX_MEM` | `--dns-cache-max-mem` | DNS cache memory cap (e.g. `64MB`) |
| `SCONE_DNS_MAX_INFLIGHT` | `--dns-max-inflight` | concurrent DNS query cap |
| `SCONE_METRICS_ADDR` | `--metrics-addr` | metrics endpoint (`ADDR` or `off`) |
| `SCONE_METRICS_TOKEN` | `--metrics-token` | metrics endpoint bearer token |
| `SCONE_TOPOLOGY_PROFILE` | `--topology-profile` | topology profile (`tiny`/`small`/`default`/`high`/`very-high`/`devtest`; default `default`) |
| `SCONE_ANCHOR_KEY` | `--anchor-key` | anchor keyfile (checkpoint + production) |

`SCONE_ANCHOR_PASS` is NOT a variable's option: it is the DEFAULT
NAME of the **variable holding the anchor passphrase**
(`--anchor-passphrase-env`, default `SCONE_ANCHOR_PASS` at the CLI
level) — the relay reads the passphrase from the variable this
flag names.

**Anchor auto (default ON)**: without `--anchor-key`, an anchor
keyfile is generated/persisted automatically in the data dir
(`anchor.sconekey`, passphrase `SCONE_ANCHOR_PASS` if defined,
otherwise randomly generated and written to `anchor.pass`) — the
relay becomes a producer as soon as its key owns live
domains/TLDs (PoS eligibility is never free). `--no-anchor-auto`
keeps the relay strictly passive. The default env var of
`--anchor-passphrase-env` is `SCONE_ANCHOR_PASS`.

`--network`: the default follows the BUILD (mainnet on
mainnet-enabled binaries, testnet otherwise); `devnet` is the
resettable local network (own
`scone-devnet` genesis, near-zero 0..=2 bit PoW — wiping
`~/.scone/devnet/` and relaunching resets the chain). Sets the
genesis, PoW difficulties, and the default
data-dir (`~/.scone/<network>/`); a tx from another network is
refused in typed fashion (`WrongNetwork`).

The offline `scone tx` surface was removed (cli-simplify-v1):
the signed-hex format remains a web/API flow. The CLI covers
the user paths (`domain register`, `renew`, `domains`); the M8b
family (`set-tld-open`, `assign-domain`, `transfer-tld`,
`revoke-tld`) is reachable through the registrar web app.
`register` claims mine their PoW at the network difficulty
(milliseconds on devnet).

The `--geo-*`/`--health-*` options configure GeoDNS and health
checking **of the embedded DNS resolver**: full semantics,
precedence order, and the local GeoIP table are documented in
[`dns.md`](dns.md#geodns-geographic-routing) — geographic routing
is **always on** (embedded GeoIP table, no option required);
`--health-check` is off by default, classic path unchanged.

### `domain register|update` (M5) — signed transactions in one command

These commands chain build → sign (keystore) → submit →
(confirmation) in a single invocation.

- **register**: builds a signed `RegisterDomain` (timestamp = now,
  empty proof in devnet), submits it, waits for on-chain
  confirmation (sequence ≥ 0, 30 s timeout). An already registered
  name is refused client-side AND relay-side.
- **update**: the records file is the sole source of truth. The
  sequence (on-chain + 1) and the `record_hash` (canonical BLAKE3
  of the file's records) are **derived**, never entered: signer and
  verifiers cannot diverge. After the UpdateDomain confirmation,
  the `SignedDnsRecord` (records from the file + owner + Ed25519
  signature over the canonical encoding) is published to the DHT
  via `put_record` (which re-verifies against the chain). An
  unregistered domain, a non-owner identity, or a file error fail
  cleanly, without touching the chain.

#### Records file

Text, one record per line, `#` = comment, blank lines ignored; the
sorted canonical encoding makes the file's order have no effect on
the committed hash. The file is the COMPLETE set (an update
replaces, it does not merge). Types:

```text
A 192.0.2.1
AAAA 2001:db8::1
CNAME www.example.uip
NS ns1.example.uip
MX 10 mail.example.uip
TXT free text (spaces normalized)
```

Typed errors: unknown type, invalid IP/name (with line number),
empty set, duplicate, > `MAX_RECORDS_PER_SET`.

## `--genesis` (founders)

`scone relay --genesis genesis.bin` — **founder nodes only**
(kickstarter campaign; the file is produced by the `scone-genesis`
binary). Normative rules:

1. The file is rejected **before reading** if it exceeds 1 MiB,
   then strictly decoded (`decode_complete`: no trailing bytes).
   The hash of block 0 is **recomputed**, never read from the file.
2. Semantic validation: height 0, `consensus` starting with the
   requested network's network id (`--network`), no transactions,
   canonical allocation manifest. Any violation = typed error at
   startup (never a panic, the daemon does not start).
3. **Virgin** data-dir: the chain is born on the allocated genesis —
   allocations are applied in RAM **before any disk write**, then
   persisted (seeded states + genesis block under a meta key). The
   first produced block can therefore only extend the allocated
   genesis.
4. **Already productive** data-dir: the store's birth genesis is
   the `prev_hash` of block 1 (self-certifying). Equal file hash =
   silent OK (the option is **redundant after the first startup**);
   different = typed refusal — a node joins a network via sync,
   never via `--genesis`.
5. Restart **without** the option: a founder-born data-dir recovers
   its allocated genesis via the meta key (RAM rebind at every
   boot).

The constant genesis hash pin (anti network-divergence) is a
separate workstream.

### `scone-genesis` binary manifest (kickstarter)

The `scone-genesis` binary (crate `scone-genesis`) prepares the
`genesis-manifest.json` manifest (`ALLOC-V1` format) then produces
`genesis.bin`. Normative rules:

- `scone-genesis new --network <network>` creates an empty
  manifest; `add`/`remove`/`list` manage allocations, `build`
  produces `genesis.bin` (+ hash), `verify` revalidates it.
- `add domain <name>` requires the domain's **TLD to already be
  allocated in the manifest** (immediate rejection otherwise — the
  chain would have refused it at genesis application); its
  `valid_until` is **automatic**: `born_at + 1 year` (the
  manifest's default_validity), never entered by hand.
- `add tld <name>`: **open by default** (nobody's, self-service,
  1-year floor, owner = zero on the wire — kind `0x03`);
  `--private` = private (owner, genesis term `born_at +
  TLD_TERM_SECS` — 3 years, standard lifecycle: `RenewTld`,
  60-day grace).
- Canonical sort by name (equal contents → same block → same hash,
  whatever the insertion order); duplicates refused.

## Limits and memory guarantees

| Resource | Bound |
|---|---|
| P2P frame | `MAX_MESSAGE_LEN` (1 MiB) before parsing |
| Mempool | 4096 tx (typed rejection beyond); `MAX_PENDING_PER_DOMAIN = 4` pending txs per domain |
| Mempool grace | `MAX_GRACE_ATTEMPTS = 10` reinsertions of a tx flushed for a non-definitive reason (≈ 7 min at 40 s/cycle), then eviction |
| Fork parking | `PARKED_CAP = 2048` competing blocks |
| Accumulated checkpoints | `MAX_PENDING_CHECKPOINTS = 256` (eviction: smallest data hash) |
| Simultaneous DHT waiters | `MAX_DHT_WAITERS` (256); DHT lookup bounded to 30 s (`DHT_LOOKUP_TIMEOUT`) |
| Blocks per sync reply | 1 per round-trip; ≤ 10,000 batches per peer (`MAX_SYNC_ROUNDS`); protocol bound `MAX_BLOCKS_PER_REQUEST = 128` on the request |
| DHT record | `MAX_DHT_CACHE_ENTRY` (64 KiB) |
| Persistent DHT cache (disk) | double LRU bound: `DHT_CACHE_MAX_ENTRIES` (100,000 entries) AND `DHT_CACHE_MAX_BYTES` (2 GiB) — whichever hits first evicts by key order; GC eviction in the SAME transaction as the block (a domain killed by the chain GC never leaves an orphan entry behind) |
| DHT negative cache | `DHT_NEGATIVE_TTL_SECS` (30 s), FIFO cap `DHT_NEGATIVE_CAP` (4,096) |
| Chain in RAM | genesis + tip window (history served from the store) |

Anti-loop rule (H2): a transaction is only re-broadcast if it was
**actually inserted** into the mempool (first sight); a duplicate
(already in the pool or already in the chain) is never re-relayed.
An already-canonical block is detected and ignored before
`push_block` (no re-store, no re-broadcast).

Open decisions: persistent network identity of the relay, broadcast
gossip (currently: unicast request-response to each known peer).
Resolved since the initial writing: real fork choice (parking +
reconsider_parked, see above), PoW consensus (signed producer +
anchor committee), DNS resolver (the `--dns` surface, see
dns.md), control channel (RPC→p2p unification: no more JSON
TCP).

## Graceful shutdown (SIGTERM / SIGINT)

The `scone relay` daemon (foreground) catches SIGTERM and SIGINT
via `tokio::signal`: on the signal, the `Relay::run` future is
cancelled (`tokio::select!` in the binary), which drops the libp2p
Swarm and closes the QUIC sockets. The tokio runtime is then
stopped with a bounded shutdown (≤ 2 s) that drops the detached
DNS tasks (UDP/TCP) and closes their sockets — the goal is that a
`kill` leaves no orphaned UDP socket (case observed on testnet 60).
A signal-driven shutdown logs `relay shutting down (signal)` (INFO,
stderr) and exits with exit code **0**: a requested shutdown is not
a failure. If handler installation failed, the signal's default
behavior applies (degradation, never a panic).


## Tor transport (open issue)

QUIC/UDP does not traverse plain SOCKS proxies, so relays cannot
trivially run over Tor today. A native libp2p Tor transport
(onion-addressed relay) is the tracked path forward — see the
website's censorship-resistance guide for the current state and
the `.tor` CNAME bridge (readable names for hidden services,
supported today).

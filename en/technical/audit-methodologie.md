# Security Audit Methodology — Scone (pre-mainnet)

Working document. Target: confidence for production deployment.
Cross-crate references: the hard rules of AGENTS.md (no homegrown
crypto, strict decoding, recomputed hashes, determinism) are the
audit's requirements specification.

## 0. Scope and threats (before any line of code is re-read)

Adversary model: an arbitrarily hostile p2p node (can send any
bytes over QUIC/UDP), a hostile tx submitter, a curious relay
operator, a stolen data-dir. EXCLUDED by design: the end client
(DNS is public by nature), global availability (chaos already
tested).

Asset table, by criticality order:
1. **Ownership** (state chain: Register/Update/Transfer/Assign) —
   a false positive = domain theft.
2. **Signature/key possession** (keystore, Ed25519, OwnerId
   derivation).
3. **Untrusted input validation** (wire decoding, limits,
   panics).
4. **Resource DoS** (DNS cache, mempool, DHT, sync).
5. **Determinism/consensus** (reorg, GC, state resurrection).

## 1. Automated static audit (day 1 — mechanical)

- `cargo audit` (dependency CVEs: libp2p/quinn/rustls/redb/ed25519).
- `cargo geiger` + review of `unsafe` (expected ≈ 0).
- `cargo clippy --workspace --all-targets -- -D warnings`
  (already green, keep as a CI barrier).
- Fuzzing the decoding boundaries (days 2-3):
  `cargo-fuzz` on `scone_protocol::decode_complete` (Message,
  Transaction, Block, SignedDnsRecord, Checkpoint) — this is THE
  surface where "never panic on untrusted data" must hold.
  Budget: 24 h of continuous fuzzing, corpus seeded from existing
  tests. Exit criterion: zero crashes, zero timeouts, coverage of
  decoding branches > 95%.
- Fuzzing the DNS wire parser (dns.rs `parse_query`,
  `extract_answers`) and the record file parser (domain.rs).

## 2. Targeted manual review per component (days 4-8)

Ordered by asset criticality, 1 component = 1 checklist:

### 2.1 scone-crypto + identity derivation (day 4)
- OwnerId = BLAKE3(pubkey): check byte canonicalization (no
  ambiguous representation → 2 keys, 1 owner).
- Ed25519: strict DESERIALIZATION (non-canonical key rejected?),
  verification with context separation (each use has its domain
  tag: SCONE-TX-SIG-V1, SCONE-BLOCK-V2, DHT record — no
  cross-protocol replay possible).
- Keystore: passphrase → key (KDF? iterations?), zeroization,
  file permissions, padding oracle on the ciphertext.

### 2.2 Transaction application (day 5) — the core
For EACH Transaction variant (13 today):
- Is the tx idempotent under replay (unique txid, nonce/sequence)?
- Does the ownership check precede any effect? (signer == owner
  learned on-chain, never trusted from the tx itself.)
- UpdateDomain sequence: exact current+1 (not ≥) — check
  rejection of gaps and rejection of replays.
- Renew: time bounds (≤ 3 years, > now) — time comes from the
  BLOCK, never the local clock (rule already set, check every
  call site).
- AssignDomain/TransferTld: is the assignee's pubkey LEARNED
  (the pubkey-less pool trap — cf. skill) on all paths.
- GC/expiration: no resurrection possible (the
  precheck-OK/lookup-OK/push-fails paradox) — check
  gc_expired_journaled.

### 2.3 Network layer (day 6)
- Every remote byte goes through a strict bounded decoder BEFORE
  allocation (MAX_* of limits.rs) — look for unbounded
  `Vec::with_capacity(input_len)`.
- requery/sync GetBlocks: bounds per peer and per cycle
  (already: 128 blocks/req) — check for absence of re-download
  loops.
- Gossip Record: chain re-verification BEFORE cache/gossip
  (verified by my record_dht tests) — also check the fanout cap.
- kad DHT: can a hostile peer force storage of N records (per-
  peer quota?) or targeted eviction of honest records?

### 2.4 DNS surface (day 7)
- Cache: injection impossible (canonical (qname,qtype) key — no
  case/encoding collision), hard memory budget (tested), no cache
  poisoning via the fallback path (TXID + question echo — verify
  the upstream cannot be spoofed remotely).
- Answers: honest AA, honest TC, no amplification (bounded
  response/request ratio — MAX_INFLIGHT_UDP already there, check
  TCP replication).

### 2.5 Storage & genesis (day 8)
- redb: disk corruption → typed error, no panic (the diskfail
  test exists on the signer side — the relay equivalent?).
- Genesis: hash covers the whole manifest; two different
  geneses = data-dir refusal (tested); verify the impossibility
  of injecting a genesis AFTER the fact on a synced data-dir.

## 3. Adversary tests (days 9-12) — write what is missing

For each finding of the manual review, a test that proves it:
- hostile node in the sim (SimNet with a node sending malformed/
  disagreeing messages) — extend SimNet with a "passive
  byzantine" role;
- real 30-node mesh with 3 hostile nodes (random bytes, bursts of
  invalid txs, forged DHT records) — the chaos battery exists,
  add the attacker role;
- cross-checks: two implementations of the decoding (the
  protocol.md doc is NORMATIVE — an auditor implements the wire
  from the doc ALONE and compares bytes; any divergence = doc OR
  code bug).

## 4. External audit (if budget) — after 1-3

Minimal profitable scope for a vendor: scone-crypto,
scone-protocol (wire), tx application (state.rs), ~8-10 k LOC.
Format: 2 weeks, 1 senior crypto auditor + 1 reviewer. Deliverable:
report with severity, PoCs, reviewed fixes. Rely on the existing
normative documentation (docs/technical) as the basis of the
requirements specification — this is a real cost advantage.

## 5. Exit criteria (mainnet go/no-go)

- [ ] cargo-fuzz 24 h without crash on the 5 decoders
- [ ] Review 2.2 complete: each tx variant has its sheet
      (ownership/replay/time/effects) signed off by the reviewer
- [ ] 30-node mesh including hostile ones: 48 h without tip
      divergence or cache corruption
- [ ] Doc/code divergence: zero (verified by the dual wire
      implementation)
- [ ] Blocking CVEs: zero (cargo audit)
- [ ] External audit of the crypto/wire/tx scope done (if
      budgeted)

## Recommended execution order

1→2.1→2.2 (the core, 60% of the risk)→2.3→2.4→2.5→3 in parallel
with each review (the adversary test is written the day of the
finding, not after).

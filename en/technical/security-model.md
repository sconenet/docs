# Security model — committee selection and anti-grinding

> Normative document. Any doc/code divergence = bug.

## Overview

The committee of epoch N+1 is drawn from the seed
`next_seed(signing_hash(checkpoint N)) = blake3("SCONE-SEED-V1" || signing_hash(checkpoint N))`,
over the **eligibility pool frozen at the block finalized by
checkpoint N**. `signing_hash` is the hash of the checkpoint's
**content** (epoch, height, block, prev, state root — signature set
excluded): the seed is a pure function of WHAT was finalized, never
of WHICH quorum subset signed it — two views holding distinct (both
valid) subsets of the same checkpoint elect the same committee.

## Threat: grinding the committee seed

An attacker wants to influence the committee's composition. Three
attack windows exist; all three are structurally closed:

### 1. Before the seed — impossible to grind one's position

The seed of epoch N+1 derives from the **hash of checkpoint N**,
quorum-signed (floor(2n/3)+1 of committee N — `quorum_for`). Checkpoint N commits the
`state_root` of a state that **fixes the pool** (PoS base = exact
state at the finalized block, `FinalizedBase.pool`). Producing a
different seed would require a different quorum-signed checkpoint
— that is, corrupting ≥ 1/3 of committee N, which is the fault
hypothesis excluded by the BFT model.

### 2. After the seed — too late to join

Any registration (domain/TLD claim, appearance of a new owner key)
only counts for subsequent epochs, whose seeds are not yet known.
The pool is frozen in `FinalizedBase` BEFORE anyone (including the
signers of checkpoint N) can compute the seed of N+1.

### 3. "Best-of-m" selection without advantage

A key's position in the draw =
`blake3("SCONE-COMMITTEE-V1" || seed || pk)` (uniform), whose only
degree of freedom is `pk`. A grinder who generates m keys and
registers only the best-positioned one must first **know the
seed** — but at that point the pool is already frozen.
Registering m candidates "just in case" costs m proofs of work
(one per domain/TLD) and yields exactly m/|pool| expected presence:
**no advantage over m direct registrations**. The adaptive
strategy (see the seed, then choose what to register) is
temporally impossible.

### 4. Last-producer bias (withholding attack)

The producer of the last block before a checkpoint could withhold
its block to influence `signing_hash(checkpoint N)` and hence the seed.
Two safeguards:

- the checkpoint signs a state at height H, and the **pool is
  evaluated at the timestamp of block H** — withholding a block
  delays the checkpoint but does not change the pool of an epoch
  already frozen by checkpoint N−1 (pools overlap little from
  epoch to epoch);
- withholding costs the attacker its own production reward and
  only affects LIVENESS (delayed checkpoint), never SAFETY: two
  incompatible final checkpoints would require ≥ 1/3 double
  signatures, provable by `SlashTx` and punished by banning.

Conclusion: a VRF or commit-reveal would add nothing here — the
seed is already a function of a quorum-signed event that is
unpredictable-delayable-but-unforgeable, and the pool freeze
removes the only adaptive lever. Do not complicate the protocol
(constraint P1.7: "if necessary" — analysis: not necessary).

## Equivocation and slashing

An anchor that signs two conflicting checkpoints at the same epoch
can be proven by anyone holding both signatures (`SlashTx`,
discriminant `0x15`): banishment from the PoS pool (journaled,
reversible on reorg). The accused's domains/TLDs are not seized
(minimal sanction — M9 decision).

## Passive acquisition and PoS eligibility

**Question**: can a third party impose a possession on me — and thus a
PoS obligation — without my consent?

**Yes for the possession, no for the debt.** A name can be acquired
passively: cession transactions (`TransferTld`, `TransferDomain`) are
signed by the **transferor** alone (the recipient is only an opaque
`OwnerId`, never consulted), `AssignDomain` is signed by the TLD owner,
and genesis allocations land on the manifest's key. This is an assumed
feature: gifts, dead drops, bequests. The anti-Sybil property does not
depend on it: one seat per **distinct owner of a live name**
(`owner_refcounts`), and every possession costs someone work.

**No imposable debt**: slashing punishes only **proven equivocation**
(two conflicting signatures, `SlashTx`, banning); a passive key signs
nothing and can therefore never be slashed. The absence of a drawn
member goes to **recovery** (re-draw), without sanction. A seat
received unwanted remains disposable: expiration + GC, `SetTldOpen`
(the seat is released upon opening), `RevokeTld`, or re-transfer.

**Tracked decision — no acquisition consent**: a two-phase claim
(pending transfer, acceptance signed by the recipient) was considered
and rejected — it breaks dead drops and "first applied wins", adds a
state machine over names, and protects against a demonstrably
non-existent risk (no imposable debt, see above).

## Properties verified by tests

- `checkpoint_hash_engages_signatures`: the checkpoint hash
  covers the signatures (no post-hoc tampering);
- `seed_chain_and_recovery`: the seed chain derives exactly,
  recovery k included;
- `small_committee_never_finalizes`: BFT guard — a committee
  below `MIN_FINALITY_COMMITTEE_SIZE` never finalizes;
- e2e 4 relays: a checkpoint finalizes and propagates, same epoch
  on all nodes.

## Replay attacks on delegated authority (counter bindings)

Five transactions mutate WHO may act on a namespace: `TransferTld`,
`TransferDomain` (ownership), `SetRecordDelegates` (record-delegate
keys), `SetTldOperator` / `SetDomainOperator` (operation delegates).
A recorded transaction of these flavors stays cryptographically valid
FOREVER (the signature covers static fields; the cession PoW solves a
challenge derived from the namespace itself). The only thing that
ages out is the bounded replay window (last 256 blocks). Beyond it,
re-arm analysis:

- **Cession A→B→A**: A transfers to B, B transfers back to A — the
  state returned to a former configuration. Replaying the ORIGINAL
  A→B transaction re-executes the transfer without consent (the
  owner check passes again; a static-challenge proof still solves).
- **Delegate-set cycle**: delegates set to `{9,10}`, cleared to `∅`,
  then ANY transaction that restores `{9,10}` — a replay of the
  first `SetRecordDelegates` resurrects revoked keys.

A binding computed as a pure function of CURRENT state (owner, set
contents, ...) returns to its old value exactly when the state does —
it would re-arm the very replay it must kill. The defense is a
MONOTONIC generation: every namespace carries `mutations: u64`,
incremented by each of the five guarded transactions; a transaction
binds to the counter of the generation it was built on
(`BLAKE3-256("SCONE-BINDING-MUTATIONS-V1" ‖ id ‖ generation ‖
counter)`) and application verifies that binding against the CURRENT
counter. Counters never return; therefore no replay re-arms. The
generation marker (`registered_at` for domains, `valid_until` for
TLDs) keeps previous LIVES of a re-registered namespace disjoint.

Residual edge (documented): the binding does not depend on the
signer's KEY. All five guarded transactions are signed by the
namespace's CURRENT owner, so a cross-life replay additionally
requires the same owner to hold the namespace in both generations
and the counter to re-attain the recorded value — a self-controlled
scenario for delegates/operators, and an owner-victim case for
cessions that requires the victim to have re-claimed their own dead
namespace years later. Accepted pre-mainnet; revisit if transfer
volume or namespace recycling makes it material.

## Known limits (documented, unfixed)

- The eligible pool includes owners of live TLDs (claim = PoW
  spent) — Sybil surface bounded by the PoW cost per key;
- The checkpoint's `state_root` is verified in RAM only when the
  checkpoint targets the tip; retroactive replay is a
  storage/relay path (see `docs/technical/blockchain.md`).

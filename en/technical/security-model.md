# Security model — committee selection and anti-grinding

> Normative document. Any doc/code divergence = bug.

## Overview

The committee of epoch N+1 is drawn from the seed
`next_seed(hash(checkpoint N)) = blake3("SCONE-SEED-V1" || hash(checkpoint N))`,
over the **eligibility pool frozen at the block finalized by
checkpoint N**.

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
its block to influence `hash(checkpoint N)` and hence the seed.
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

## Properties verified by tests

- `checkpoint_hash_engages_signatures`: the checkpoint hash
  covers the signatures (no post-hoc tampering);
- `seed_chain_and_recovery`: the seed chain derives exactly,
  recovery k included;
- `small_committee_never_finalizes`: BFT guard — a committee
  below `MIN_FINALITY_COMMITTEE_SIZE` never finalizes;
- e2e 4 relays: a checkpoint finalizes and propagates, same epoch
  on all nodes.

## Known limits (documented, unfixed)

- The eligible pool includes owners of live TLDs (claim = PoW
  spent) — Sybil surface bounded by the PoW cost per key;
- The checkpoint's `state_root` is verified in RAM only when the
  checkpoint targets the tip; retroactive replay is a
  storage/relay path (see `docs/technical/blockchain.md`).

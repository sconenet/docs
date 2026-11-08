# Fuzzing — hostile-input robustness

The network delivers hostile bytes: any peer can send anything over
QUIC/UDP, a disk can hold corrupted files, a keyfile can be
truncated mid-write. Every decoder of untrusted input must satisfy
one invariant (SECURITY_AUDIT §10, audit-methodologie AM2):

- **never panic**,
- **never allocate unbounded memory**,
- **never hang**.

A typed rejection (`Err`) is the expected outcome — it is also the
coverage proxy the harness reports. The fuzz infrastructure asserts
this invariant continuously instead of trusting code review.

## The two layers

| | `fuzz-smoke/` | `fuzz/` |
|---|---|---|
| Tool | std-only deterministic mutation runner (in-house) | cargo-fuzz, libFuzzer + AddressSanitizer |
| Toolchain | stable — never blocks on nightly | nightly + g++ required |
| Workspace | member (`cargo test -p fuzz-smoke`) | **outside** the workspace by design |
| Role | CI smoke + operator long runs | deep coverage-guided campaigns |

### Layer 1 — `fuzz-smoke/` (deterministic)

Each target is a plain `fn(data: &[u8])` calling the REAL
production decoder, wrapped in `catch_unwind`. Any panic = decoder
bug: the offending input is saved under
`fuzz-smoke/artifacts/<target>/` and the run exits non-zero. The
PRNG is a seeded SplitMix64: **the same seed replays the exact same
input sequence** — a failure is always reproducible.

`cargo test --workspace` runs the smoke battery: 10,000
deterministic iterations per real target (fixed per-target seeds),
the planted-panic scenario (catch + artifact save + continue), and
the determinism contract (same seed → same sequence). The LONG runs
(≥ 50k, e.g. 1M iterations/target) are the operator's manual
pre-mainnet step.

### Layer 2 — `fuzz/` (coverage-guided)

The cargo-fuzz migration is deliberately mechanical: every target
is already a ready `LLVMFuzzerTestOneInput`, and the built-in
corpora are reusable as libFuzzer seed dirs. It lives outside the
workspace so nightly/ASan requirements never leak into CI. Audit
budget: 24 h of continuous fuzzing; exit criterion — zero crashes,
zero timeouts, decoder branch coverage > 95 %.

## Targets

| Name (`--target`) | Production surface under test |
|---|---|
| `wire` | `decode_complete::<Message>` — the wire frame decoder |
| `dns_query` | `parse_query` — the DNS UDP/TCP query parser |
| `dns_answers` | `extract_answers` — DNS answer extraction |
| `record_file` | `scone::record_file` — the CLI record-file parser |
| `keyfile` | `.sconekey` bytes parse/open |
| `panic_scratch` | harness canary — panics on purpose to prove the artifact-capture path works |

`--target all` covers the five real decoders (the canary is opt-in).

## Running

```bash
# list the targets
cargo run -p fuzz-smoke -- --list

# short run (debug build, default 10k iterations)
cargo run -p fuzz-smoke -- --target wire

# all five decoders in one go (release)
cargo run -p fuzz-smoke --release -- --target all --iters 50000

# the pre-mainnet operator run — one target, long
cargo run -p fuzz-smoke --release -- --target wire --iters 1000000

# fixed seed + custom artifacts location
cargo run -p fuzz-smoke --release -- --target wire --seed 0xC0FFEE \
  --artifacts-dir /tmp/fuzz-out
```

## What a failure looks like

A crash saves the exact offending bytes to the artifacts directory
(default `fuzz-smoke/artifacts/<target>/`) and the runner exits
non-zero. Reproduce from the artifact (corpus input) or from the
seed. Everything else — `Err(...)` typed rejections — is the
decoder working as designed and is reported as coverage, never as
a failure.

# CLI (`scone`) — logging and verbosity

> Normative behavior of the `scone` binary's logging. Any doc/code
> divergence = bug (fix the source of truth).

## Principles

- **stdout is reserved for command output** (`scone status`,
  `identity list`, `status`, …). A script can redirect stdout
  without ever finding logs in it — piping stays reliable. Accepted
  exception: `scone relay` emits the data lines
  `p2p: <multiaddr>`, `dns: <addr>`, and `metrics: <addr>` on
  stdout as soon as the listeners are bound
  (usable data for `--bootstrap` / configuration, not logs;
  emitted exactly once each, flushed immediately).
- **All logs go to stderr** (compact `tracing` format), including
  the relay's — the relay already logged to stderr before the
  migration.
- The CLI's user-facing error messages (`scone: <error>`) stay on
  stderr: this is UX, not logs. A failing command also prints a
  second stderr line, `[Failed: <error>]`, mirroring the status
  lines of successful commands.

## Levels and `--verbose` / `-v`

| Invocation | Effective level |
|---|---|
| `scone <cmd>` (without `-v`) | `warn` |
| `scone relay` (without `-v`) | `info` |
| `scone -v <cmd>` / `scone relay -v` | `info` |
| `scone -vv …` | `debug` |
| `scone -vvv …` (and more) | `trace` |

`-v`/`--verbose` is a **global and repeatable** flag: it is
accepted before or after the subcommand (`scone -vv identity list`
and `scone identity list -vv` are equivalent).

**Why `scone relay` stays at INFO by default**: the relay is a
daemon — it must log its activity (p2p listening, accepted and
produced blocks, connected peers, rejections) to be operable,
without requiring a flag. One-shot commands, on the other hand,
stay silent (WARN) so as not to pollute a terminal or a script.

## `RUST_LOG` (override)

If the `RUST_LOG` environment variable is set, it **entirely
replaces** the level derived from `-v` (`env-filter` syntax):

```bash
RUST_LOG=debug scone identity list        # debug without -v
RUST_LOG=error scone -vv relay            # error wins over -vv
RUST_LOG=scone_network=trace,info scone relay   # per-target
```

## Environment configuration (`.env`)

Before parsing arguments, `scone` loads a `.env` file from the
current directory (`dotenvy::dotenv()` in `main.rs`). Missing
file = silent no-op, never an error. The loaded values become the
baseline of the relay options that declare an environment
variable (`env = "…"` in `cli.rs`).

Precedence (one option, three sources):

```text
explicit CLI flag  >  .env variable  >  built-in default
```

- a flag passed on the command line ALWAYS wins;
- otherwise the variable (whether loaded from `.env` or set in the
  real process environment — same variable, same effect);
- otherwise clap's built-in default.

`.env` holds ONE value per variable: dotenvy applies the FIRST
occurrence of a variable (duplicates are silently skipped) and
never accumulates; the real process environment also wins over
the file. Multiple bootstrap seeds or DNS upstreams therefore
require repeated `--bootstrap` / `--dns-upstream` flags on the
command line.

The complete table of `SCONE_*` variables (network, P2P listen,
bootstrap, genesis, UPnP, DNS surface, metrics, topology
profile, anchor) and each one's semantics lives in
[`relay.md`](relay.md#environment-configuration-env); the
commented template is `.env.example` at the repository root (copy
to `.env`, never commit it — `.env` is gitignored).

## Level semantics (code convention)

- `error!`: fatal local failures (signer disabled on an
  unreadable/corrupted signer state, unusable store, deep-fork
  recovery failure, anchor passphrase not persisted) — network
  failures are swallowed as `warn!` and only fatal local failures
  surface as a typed error from `run()`;
- `warn!`: hostile or malicious network entries turned away,
  stale txs evicted from the mempool, conflicting txs dropped
  during production, outbound/inbound failures, DHT key mismatch;
- `info!`: block accepted/produced, p2p listening, peer
  connected, relay startup (state checkpoints) and the build
  flavor line (`build: mainnet-enabled` / `build: standard`);
- `debug!`: internal details (tx accepted, peer disconnected, RPC
  round trip, keystore listing, kad events under
  `SCONE_KAD_DEBUG`);
- `trace!`: reserved for payloads (no call sites today).

Library crates (`scone-network`, …) emit `tracing` macros but
**initialize no subscriber**: initialization lives only in the
`scone` binary (before subcommand dispatch — the relay needs its
logs from boot).


## Simple usage (cli-simplify-v1)

- **No `--peer` needed**: commands talk to the local relay by
  default; `SCONE_PEER` env or `--peer` overrides.
- **No `--identity` needed** when the keystore holds a single
  identity (it is used silently; several → the error lists them).
- **STRICT: an identity IS required for `domain register`** — an
  empty keystore fails with a typed error pointing at
  `scone identity generate` (a key minted on the fly would orphan
  the domain at the first admin operation — the seed shown once is
  not a keystore). To claim FOR another key without any keystore:
  `--to <pubkey>` (pow-to-owner — no signature, the proof is the
  whole authorization). (strict-cli-lifecycle)
- Removed: `show`, `submit`, `lookup`, `owned` — no stubs; clap
  rejects these names with its generic "unrecognized subcommand"
  error plus a closest-name suggestion.
- New (then merged — `domain register` renews): `scone domains`, `scone domain
  get|set` (single record, no file).

```bash
scone identity generate me
scone domain register example.scone   # claim OR renewal (unified)
scone domain set example.scone A 192.0.2.1
```

`domain register` on a name YOU OWN runs the renewal: a **fixed
one-year term** from the later of the current expiry and now —
clamped to the anti-hoarding maximum (`now + 3 years`) for a
domain stacked near the ceiling. Renewal durations are FIXED: no
choice. The extension is lossless (renewing early loses
nothing) — no gate: yours = renewal runs, announced. The
`domain renew` verb is gone (unify-lifecycle-commands).

## TLD lifecycle (cli-unified-lifecycle)

`scone tld register <tld>` is the one-command TLD entry point,
symmetric with `scone domain register` — the on-chain state is
read FIRST, then the right action runs:

- **free TLD** → registered, born **OPEN by default** (no flag:
  self-service namespace, 1/3 of the PoW). `--private` opts into
  an owned namespace at the full PoW price, with an explicit
  confirmation (economic decision). `--yes` skips confirmations
  (scripts).
- **yours, expiring within 90 days** → the RENEWAL is proposed
  (PoW bound to the current expiry — anti-reuse) and runs after
  confirmation.
- **yours, far from expiry** → typed error (exit non-zero): the
  TLD term is FIXED (`now + 3 years`) — early renewal would LOSE
  the remaining time, so early renewal is not offered (the gate
  protects, it does not complicate). The error carries the hint
  « renewal opens within 90 days of expiry » (strict-cli-lifecycle:
  a silent exit 0 would let a script believe it renewed).
- **someone else's** → clean typed error with the owner.

```bash
scone tld register mytld            # open birth (default)
scone tld register mytld --private  # owned namespace (confirm)
```

Likewise, `scone domain register <name>` on a name you own runs
the renewal (fixed one-year term) instead of failing.

## Record delegates (KSK/ZSK-lite)

A domain's owner can grant OTHER keys the right to sign
`domain update` on its behalf — the record-delegate set (at most 3
keys, full replacement on every `set`):

```bash
# grant (owner-signed; the CLI fetches the namespace's CAS
# generation and binds the transaction to it):
scone domain delegates set example.uip \
  --keys <pk-hex-1>,<pk-hex-2> --identity alice

# inspect (no identity needed):
scone domain delegates list example.uip

# revoke everything:
scone domain delegates clear example.uip --identity alice
```

A delegate then signs `domain update` exactly like the owner would
(`--identity deleg`) — on-chain the tx's `owner` field is the
DOMAIN's owner while the signature comes from the delegate. The
grant is owner-exclusive (a delegate cannot manage the set), cleared
by a transfer, and every `set`/`clear` bumps the namespace's
mutation counter (CAS anti-replay — see `transactions.md`): a
replayed `set` cannot resurrect revoked keys. A revoked delegate's
update is refused locally ("neither the owner nor a record
delegate").

## Advanced commands

The defaults above cover everyday use. For scripts and web/API
interop the full surface stays available — every action has exactly ONE
canonical path; the former root aliases (`show`, `lookup`,
`owned`, `renew`) were removed (remove-cli-aliases); `domain
info` remains as a compatibility alias of `domain get`.

The offline hex surface (`tx build|sign|verify`, `submit tx`
subcommands) was REMOVED (remove-tx-surface-v2): the M8b
operations it served are first-class commands now —
`scone tld open <tld>` (one-way self-service opening, opening
PoW mined for you), `scone tld transfer <tld> <owner>`,
`scone tld revoke <tld>`, `scone domain assign <name> <owner>`
(TLD owner, private TLD) and `scone domain transfer <name>
<owner>`. Every one: signature, submission, txid. The transfer
paths (`tld transfer`, `domain transfer`) pre-read the on-chain
state client-side — the cession proof is bound to the current
generation — while `tld open`, `tld revoke` and `domain assign`
do no client-side state read (the node enforces). DNS record
resolution moved to `scone domain get <name>` (ex-`record get`,
ex-`domain resolve`); the hex publication channel (`record put`)
is gone —
editing goes through `domain set` / `domain update`.
- `scone domain mine <name>` — mines ONLY the registration
  proof (no identity, no keystore, no submission): prints the
  `scone-domain-pow-v1` JSON `{type, name, network_id, epoch,
  proof}` to stdout. Delegated mining: another machine mines
  for you, your keys never leave yours. Pipe or redirect it:
  `scone domain mine n.tld | scone domain register n.tld
  --proof -` (or `--proof FILE`). The proof is name- and
  network-bound; freshness is enforced by the node at apply
  time (epoch validity window). With `--mine-for <pubkey>` it
  instead mines a POW-TO-OWNER proof for that beneficiary and
  prints the `scone-domain-pow-to-v1` JSON `{type, name,
  network_id, epoch, pow_to, proof}` — consumable ONLY by
  `register --to` / `renew --to` for that exact key (the
  challenge embeds it; case-insensitive check at submission).
- Pow-to-owner submission (no keystore needed — `--to` is
  exclusive with `--identity`): `scone domain register <name>
  --to <pubkey>` / `scone domain register <name> --to <pubkey>`
  and `scone tld register <tld> --to <pubkey>` / `scone tld
  renew <tld> --to <pubkey>` submit a proof whose challenge
  binds the BENEFICIARY key; the on-chain `owner` is recomputed
  as `hash(pubkey_dest)` (a renew NEVER changes hands). The
  `--to` commands always mine their proof in place — `--proof`
  exists ONLY on `domain register --to`: with `--proof
  -`/FILE it consumes a `scone-domain-pow-to-v1` payload and
  refuses a proof mined for ANOTHER beneficiary. Pipe style:
  `scone domain mine n.tld --mine-for <pubkey> | scone domain
  register n.tld --to <pubkey> --proof -`.
- `scone domain export <name> [--out FILE]` — writes the
  chain-verified record set in the exact `update --file` format
  (round-trip guaranteed: export → edit → update re-submits the
  same set). Without `--out`: stdout. `domain get` renders the record
  part in the same readable format, so `domain get <name> >
  zone.txt` is directly importable (`--raw` appends the hex
  encodings). Kinds with no textual syntax (GEOROUTE, unknown)
  are exported as commented warnings — never silently dropped.

`domain get` also prints the `domain_id` and `valid_until`.

## PoW benchmark (`scone bench-pow`)

Measures the machine's Argon2id hash rate (the same hash the
consensus verification uses) and translates the network's
difficulties into concrete waiting times:

```
scone bench-pow --network mainnet [--seconds 5] [--cores N]
```

Output: total and per-worker H/s, then the expected average time
per operation — transfer/assign (flat rate), private-TLD opening,
and the TLD/domain registration floors (the length curve). The
selected network determines the Argon2id parameters (mainnet:
64 MiB, t=2 — the real cost; testnet/devnet: the light profile,
millisecond proofs). Each worker allocates its own Argon2id lane
— 64 MiB on the mainnet profile, 8 KiB on the light
testnet/devnet profile — size `--cores` to the available RAM.

## Builds: dev vs mainnet-enabled (mainnet-compile-gate)

The relay startup log opens with the build flavor
(`build: mainnet-enabled` / `build: standard`) — the runtime
counterpart of the `+mainnet` version marker.

Default builds carry **no mainnet**: `scone relay --network
mainnet` (and registrar services with `SCONE_NETWORK=mainnet`)
fail immediately with a rebuild hint. The official distributed
binary is built with:

```bash
cargo build --release --features mainnet
```

`scone --version` shows `+mainnet` on such builds.

The default `--network` follows the build too: `mainnet` on
mainnet-enabled builds (the official binary lands users on the
production network), `testnet` otherwise. An explicit
`--network` flag or the `SCONE_NETWORK` env always wins
(clap order: flag > env > default). This is an
operational guardrail (accidental runs, dev/CI binaries) — not
a cryptographic boundary; consensus rules remain the defense
against hostile nodes.

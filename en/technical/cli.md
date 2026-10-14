# CLI (`scone`) — logging and verbosity

> Normative behavior of the `scone` binary's logging. Any doc/code
> divergence = bug (fix the source of truth).

## Principles

- **stdout is reserved for command output** (`scone show`,
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
  stderr: this is UX, not logs.

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
daemon — it must log its activity (RPC/p2p listening, accepted and
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

Repeatable variables (`SCONE_BOOTSTRAP`, `SCONE_DNS_UPSTREAM`)
accumulate: one line per value in `.env` (dotenvy appends).

The complete table of `SCONE_*` variables (network, P2P listen,
bootstrap, genesis, mDNS, UPnP, DNS surface, metrics, topology
profile, anchor) and each one's semantics lives in
[`relay.md`](relay.md#environment-configuration-env); the
commented template is `.env.example` at the repository root (copy
to `.env`, never commit it — `.env` is gitignored).

## Level semantics (code convention)

- `error!`: failures (no call sites today — network failures are
  swallowed as `warn!` and only fatal local failures surface as a
  typed error from `run()`);
- `warn!`: hostile or malicious network entries turned away,
  stale txs evicted from the mempool, conflicting txs dropped
  during production, outbound/inbound failures, DHT key mismatch;
- `info!`: block accepted/produced, RPC/p2p listening, peer
  connected, relay startup (state checkpoints);
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
- Removed (stubs suggest the replacement): `show`, `submit`,
  `lookup`, `owned`, `record put`.
- New: `scone renew <name>`, `scone domains`, `scone domain
  get|set` (single record, no file).

```bash
scone identity generate --name me
scone domain register example.scone
scone domain set example.scone A 192.0.2.1
scone renew example.scone
```

## Builds: dev vs mainnet-enabled (mainnet-compile-gate)

Default builds carry **no mainnet**: `scone relay --network
mainnet` (and registrar services with `SCONE_NETWORK=mainnet`)
fail immediately with a rebuild hint. The official distributed
binary is built with:

```bash
cargo build --release --features mainnet
```

`scone --version` shows `+mainnet` on such builds, and the
relay startup log says `build: mainnet-enabled`.

The default `--network` follows the build too: `mainnet` on
mainnet-enabled builds (the official binary lands users on the
production network), `testnet` otherwise. An explicit
`--network` flag or the `SCONE_NETWORK` env always wins
(clap order: flag > env > default). This is an
operational guardrail (accidental runs, dev/CI binaries) — not
a cryptographic boundary; consensus rules remain the defense
against hostile nodes.

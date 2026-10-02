# UDP + TCP DNS Server (M6 + M7b) — `scone dns`

> Normative specification of the Scone DNS surface.
> Any doc/code divergence = bug (fix the source of truth).

## Overview

The **DNS server** is an optional surface of the relay: it answers
**UDP and TCP** DNS queries (RFC 7766, same port) for Scone names
**only from data verified against the chain**, and offers an
**optional recursive fallback** (`addr:port`) for everything else.
It owns neither chain, nor store, nor DHT: resolution goes through
the relay (process), via the local control RPC — the same verified
path as the CLI.

```text
              UDP+TCP :5353 (default 127.0.0.1)
                    │
             scone relay --dns 127.0.0.1:5353
                    │  ┌────────────────────────────────┐
                    ├──┤ Bounded cache (qname,qtype)    │
                    │  └────────────────────────────────┘
                    │  Scone-shaped name?
                    │  ├─ YES → apex search by suffix
                    │  │        → `resolve_local` RPC (relay)
                    │  │           chain + verified record (hash,
                    │  │           owner, sequence) → answers
                    │  │        apex unknown across the whole chain → NXDOMAIN
                    │  │        apex without a published record → NODATA
                    │  └─ NO → upstream fallback (if configured)
                    │           otherwise REFUSED
                    │
                    │  response budget:
                    │    UDP → 512 bytes (no EDNS); overflow =
                    │          cut at the last fitting record + TC=1 →
                    │          the client retries over TCP
                    │    TCP → 64 KiB (2-byte length prefix,
                    │          RFC 1035 §4.2.2): full set
```

## Startup

```bash
scone relay --dns 127.0.0.1:5353            # DNS surface only
scone relay --dns 0.0.0.0:53 \
            --dns-upstream 1.1.1.1:53 \
            --dns-upstream 8.8.8.8:53       # + recursive fallback
```

The relay announces the effective address on stdout
(machine-readable contract, like `p2p:`): a `dns: <addr>` line as
soon as the socket is bound.

Validation client: `scone dig <name> --dns 127.0.0.1:5353
--qtype A` (A, AAAA, TXT, MX, NS, CNAME, ANY).

## Resolution path (AUTHORITY = the chain, always)

1. **Cache**: key `(qname, qtype)`, capacity 100,000 entries by
   default (`--dns-cache-limit` to override) AND a hard memory
   budget of 64 MiB by default (`--dns-cache-max-mem`, e.g.
   `64MB`): each entry's REAL rdata size is accounted for (~128
   bytes overhead per entry) and FIFO eviction triggers on the
   first of the two bounds reached — the memory cap holds even
   against full-size entries (`MAX_DHT_RECORD_LEN`). TTL 60 s
   (authoritative) / clamped to 600 s (fallback). The negative
   cache (NXDOMAIN/NODATA) follows the same rules; each entry
   carries its origin (authoritative or fallback) so the AA bit
   stays honest on a hit. **Never cache SERVFAIL** (transient
   local failure) **nor REFUSED** (configuration state): both are
   recomputed on every query. FIFO eviction at capacity.
2. **Scone-shaped?** Strict pre-filter: labels `[a-z0-9-_]`
   (1..63), ≥ 2 labels, TLD 1..63 bytes `[a-z0-9-]` (see
   `TldName::MAX_LEN`), total length ≤ 253 — **and TLD outside
   the ICANN root** (`is_icann_tld`, frozen IANA list + RFC 2606:
   `com`, `fr`, `org`… never belong to Scone).
   A name that CANNOT be a Scone name (e.g. `www.foo_bar`,
   underscore in the TLD) goes to fallback; a possible Scone name
   (`absent.uip`) is a chain matter: unregistered → authoritative
   NXDOMAIN, even with no upstream. An ICANN TLD (`example.com`)
   is forwarded to the upstreams if configured, REFUSED otherwise
   — never resolved authoritatively by Scone.
3. **Apex search**: suffixes from longest to shortest (min.
   2 labels). The first suffix whose on-chain state is known to
   the relay = apex; the record set serves the whole qname
   (subnames included — implicit wildcard behavior, open
   decision).
4. **`resolve_local` (internal RPC)**: a single round trip, never
   a DHT waiter. On-chain state (`registered`, owner, sequence) +
   local DHT cache record **re-verified** (owner, sequence,
   record_hash == on-chain commitment). A non-conforming record
   never answers — availability ≠ authority.
   - unregistered → NXDOMAIN;
   - registered without a valid record → NODATA (NOERROR, 0
     answers);
   - verified record → answers filtered by qtype.

## Codec (RFC 1035, strict subset)

- Query: 12-byte header, **one** question, QCLASS = IN, **no
  compression** (pointer = reject). Labels lowercased and
  validated. QR=1, QDCOUNT≠1, invalid labels → **silence**
  (garbage costs nothing).
- Response: QR|RD(echo)|RA, **AA only on authoritative answers**
  (chain, fresh or cached — never on fallback nor error RCODEs),
  question copied verbatim, answers **uncompressed** (owner =
  qname).
- Types served: A, AAAA, CNAME, NS, MX, TXT, **PTR, SRV, CAA**
  (+ Unknown re-encoded verbatim; 9 native types in total, rdata
  encoded per RFC — 1035/2782/8659; strict CAA: critical bit 0x80
  only, lowercase tags). An A/AAAA query includes the set's
  CNAMEs (alias chasing). ANY query (255): the whole set filtered
  by the codec.
- Limits (M7b): **the response budget depends on the transport** —
  512 bytes over UDP (classic RFC 1035 §2.3.4 payload, no EDNS),
  65,535 over TCP. A set exceeding the budget is cut at the last
  record that fits, with **TC=1** (honest truncation, RFC 1035
  §4.2.1): the client retries over TCP (RFC 7766), where the full
  set is served. An upstream reply larger than the UDP budget is
  re-encoded truncated (TC=1) instead of being delivered as is;
  over TCP it passes byte for byte. Truncation also applies to
  cache hits (the budget is applied at response time, not at
  cache-fill time).
- RCODEs: 0 NOERROR (with or without answers), 2 SERVFAIL (local
  RPC failure), 3 NXDOMAIN (authoritative), 5 REFUSED (non-Scone
  without upstream).

## Recursive fallback

- Upstreams `addr:port` (UDP), tried in order, 2 s timeout per
  upstream, socket **connected** to the upstream (kernel
  filtering: a packet from any other source is never accepted).
  The upstream response is only accepted if it **matches the
  query**: same TXID, QR=1, question (QNAME+QTYPE+QCLASS) copied
  byte for byte. A non-conforming response → SERVFAIL, and
  nothing is cached.
- The validated upstream response is passed through as is; its
  answers are cached with origin `fallback` (AA never set, even
  on a cache hit) and TTL clamped to 600 s. Upstream error
  RCODEs (other than NOERROR/NXDOMAIN) are not cached.
- **Private by default**: no upstream = REFUSED for names outside
  the Scone shape.

## TCP (M7b, RFC 7766)

Same port as UDP (second TCP bind on the `--dns` address). The
transport follows RFC 1035 §4.2.2: every message is prefixed by
its length on **2 bytes** big-endian.

- **Query**: prefix length ≥ 1 and ≤ 512 bytes (a DNS query fits
  the classic payload — RFC 7766 §6; no EDNS served). Out of
  bounds → immediate connection close.
- **Response**: 65,535-byte budget (the 2-byte prefix maximum) —
  a set too large for UDP is served **in full** here, TC=0.
- **Pipelining**: multiple queries per connection are served
  serially, in order (one write = one prefixed response).
- **Read timeout**: 5 s per read (prefix or body) — a silent
  client is disconnected, its slot freed.
- **Concurrent connections**: 64 max (semaphore); the 65th is
  rejected immediately (connection closed), never queued — no
  unbounded growth of tasks or memory.
- Read/write/timeout errors: debug log + clean close. **Never
  panic on network input.**
- Garbage (incomprehensible query) gets silence but does not
  close the connection: the next (valid) query is served.

```text
client                     server
  │ [len u16][query]         │
  │ ─────────────────────►   │ cache → authoritative → fallback
  │ ◄─────────────────────   │ [len u16][response ≤ 64 KiB]
  │ (UDP had answered TC=1 → the client retries here)
```

## Bounds and guardrails

| Guardrail | Value |
|---|---|
| UDP datagram (query/response) | 4096 bytes (rejected beyond) |
| UDP response budget (TC=1 beyond) | 512 bytes (no EDNS) |
| Max TCP query | 512 bytes (closed beyond) |
| Max TCP response | 65,535 bytes |
| Concurrent TCP connections | 64 (semaphore, immediate reject) |
| TCP read timeout | 5 s |
| Cache | 1024 entries (FIFO) |
| Authoritative TTL | **per record** (`record.ttl`, bounds 1..=2,592,000 s; the historical constant 60 s remains only for service responses) |
| Fallback TTL (clamp) | 600 s |
| Parallel UDP queries | 256 (semaphore; beyond: drop, the client retries) |
| Timeout per upstream | 2 s |

Project rules honored: never panic on network data; strict
decoding everywhere; the server touches neither chain, nor store,
nor DHT (the relay remains the sole verification authority).

## Tests

- Unit (`scone-network::dns`): strict codec (QR, compression,
  labels rejections), authoritative answers, NODATA, NXDOMAIN,
  REFUSED without upstream, fallback to a UDP mock, bounded
  cache, TXT chunking, upstream parsing; M6 fix-up: AA-bit
  honesty (authoritative vs fallback vs errors, including cache
  hits), TC=1 truncation of an oversized set, upstream response
  rejection (TXID or question mismatch), no caching of
  SERVFAIL/REFUSED.
- M7b: per-transport budget (UDP 512 bytes → TC=1, TCP → full
  set), truncation of a cache hit, real `run_tcp` (set > 512
  served in full, pipelining of 2 queries, oversize query →
  close, silent client → 5 s timeout, garbage then valid query on
  the same connection), upstream reply > 512 re-encoded TC=1
  over UDP / byte for byte over TCP.
- Real integration (`tests/dns_server.rs`): a full relay
  (including production devnet) + registration + UpdateDomain +
  PutRecord, then real UDP queries: verified A (AA bit), TXT,
  NODATA AAAA, NXDOMAIN, fallback via mock, re-query, subname via
  apex.
- Binary E2E (`crates/scone/tests/dns_cli.rs`): the real `scone`
  binary — `relay --dns`, `identity generate`, `domain register`,
  `domain update --file`, then `scone dig` (verified A, TXT,
  NXDOMAIN, REFUSED) on a real UDP port.

## Open decisions

- No EDNS (the UDP budget stays 512 bytes; TC=1 truncation +
  TCP covers the devnet need).
- The record set carries no per-record name: the whole set serves
  every subname of the apex (implicit wildcard). Per-record
  addressing would require extending `DnsRecord` (protocol).
- The fallback forwards the raw query: a malicious upstream can
  answer anything (unvalidated response — this is the standard
  recursive model; Scone names NEVER go through it).

## GeoDNS (geographic routing, opt-in)

> Normative specification. Any doc/code divergence = bug.

### Overview

A domain can publish **geographic routing** rules in its signed
record set: `GEOROUTE` records (Scone private type `32,768`,
outside the IANA range). A resolver with GeoDNS armed
(`--geo-dns`) answers A/AAAA questions for these domains with
**the address of the endpoint chosen by the rule**, instead of a
static A/AAAA record. Everything else (domains without GEOROUTE,
other qtypes, resolvers without `--geo-dns`) follows the classic
DNS path, **byte for byte** — GeoDNS is 100% opt-in.

### GEOROUTE record (wire)

A GEOROUTE = a **scope** + ≤ 32 **endpoints** (canonical sort on
decode: ascending priority, descending weight, ascending target;
duplicate = reject):

```text
GeoScope   = 0x00 (world)
           | 0x01 continent.v (UN M49 code: AF=2, AS=142, EU=150,
                                NA=3, SA=5, OC=9, AN=999)
           | 0x02 country.v   (ISO 3166-1 numeric code: US=840,
                                TW=158, FR=250…)
           | 0x03 region.v    (OPAQUE ISO 3166-2 subdivision, u32:
                                publisher-defined number, compared
                                as is — no local table)
           | 0x04 asn.v       (AS number, u32)
           | 0x05 prefix.addr[16] || prefix.len u8
             (normalized 16-byte IP prefix: IPv4 = address in the
              first 4 bytes + ZERO tail, len ≤ 32;
              IPv6 = 16 free bytes, len ≤ 128; host bits ALWAYS
              zero — a non-normalized prefix is rejected)
GeoTarget  = 0x00 octets[4]  (binary IPv4 address)
           | 0x01 octets[16] (binary IPv6 address)
GeoEndpoint = target || priority.v || weight.v
GEOROUTE    = scope || count.v(≤32) || endpoints…
```

Validation bounds (decode): `priority > 0`, `weight ≤ 100`
(`weight = 0` = inactive endpoint, never elected), TTL within
1..=2,592,000 s like any record. A country/continent unknown to
the local tables is **allowed** on the wire: routing simply falls
back to a higher scope.

**Wire backward compatibility (v1 → v2)**: the v1 bytes are
UNCHANGED — v1 scopes (world/continent/country) keep their
discriminants `0x00..=0x02` and their exact encoding; the three
new scopes take the following discriminants (`0x03..=0x05`) in a
total order consistent with the canonical sort
(`World < Continent < Country < Region < Asn < IpPrefix`). A v1
decoder meeting a discriminant ≥ `0x03` already failed strictly
(record rejected); v1-produced bytes remain decoded identically
(see the `geo_known_bytes` test, frozen).

### Selection (resolution)

1. **Scope** — decreasing specificity, **6 levels**:
   **longest matching IP prefix → exact AS → exact country →
   exact region → continent → world**.
   - *IpPrefix*: among the routes whose prefix matches the
     client's IP (as known to the locator: `client_ip`, otherwise
     the known origin prefixes), the **longest** (`len` max)
     wins; on equal `len`, the first in canonical order (the
     order of the `routes` slice, sorted at encoding). A
     subdivision is less specific than a prefix/AS (which
     identify the exact network) and less than the country
     (opaque numbering, reliable only if the publisher and the
     locator share the same one), but more than the continent
     (it is contained in its country, hence a fortiori in its
     continent).
   - *Asn*: exact AS number of the client.
   - *Country*: exact country.
   - *Region*: exact subdivision (opaque identifier).
   - *Continent*: provided by the locator **or derived from the
     country** via the local table (`continent_of_country`); a
     country unknown to the table falls back to the explicit
     continent or world.
   - *World*.
   A v1 [`GeoLocation`] (region/asn/prefixes/client_ip unset)
   produces exactly the v1 behavior: country → continent →
   world. No matching route (including without a World scope) →
   classic DNS behavior, as if the GEOROUTE did not exist.
2. **Priority** — SRV-like semantics: only the **minimum
   priority** group is eligible; higher priorities are
   **failover** levels.
3. **Weight** — within the elected group, **deterministic
   weighted** draw: stable hash (SplitMix64) of a nonce (the
   resolver's query counter) against the cumulative weights. No
   RNG: two queries with the same nonce give the same result,
   increasing nonces distribute proportionally to the weights.
   `weight = 0` = inactive, never elected (even alone — move to
   the next level).
4. **Health failover** — an **Unhealthy** endpoint is set aside;
   if the whole minimal level is unavailable, move to the next
   priority level. **Unknown counts as available** (the unknown
   is not penalized). Everything unavailable → classic DNS
   behavior.

### Response

- A single address: **A if IPv4, AAAA if IPv6** — strict
  filtering by family according to the qtype (an A question
  NEVER answers an IPv6, and vice versa).
- TTL = that of the source GEOROUTE record (bounded at decode by
  the general TTL rules).
- AA set (authoritative response — the GEOROUTE is a verified
  signed record like any other).
- Empty response after filtering (no candidate of the right
  family, everything Unhealthy, no matching route) → the resolver
  continues on its classic path (typically NODATA: NOERROR, 0
  answers); error rcodes follow the general rules (never cached).
- A classic A/AAAA record coexisting in the set takes
  PRECEDENCE over geo (geo applies only in the absence of a wire
  answer for the qtype).

### Cache: explicit decision

The geo routing decision is **NEVER cached** under
`(qname, qtype)`: it depends on the resolver's client scope, and
such a key would serve the US route to a TW client. Geo answers
are recomputed on every query (cost: one select_scope + weighted
draw, µs) and bypass the cache. The classic (qname, qtype) cache
remains unchanged for everything else — including the NODATA
produced when geo decides nothing. Error rcodes remain never
cached.

### Health checks (`--health-check`)

- `HealthCache` per target (`GeoTarget`): `probe` = non-blocking
  read (fresh status, or Unknown), **never** a check triggered
  from the resolution path.
- Refresh: if an elected candidate is Unknown/absent from the
  cache, ONE refresh is scheduled as a background task
  (`tokio::spawn_blocking`: the TCP connect is blocking,
  forbidden in the UDP loop). Anti-spam: one refresh at a time
  per target (HashSet), restartable after the previous refresh
  finishes.
- `TcpChecker`: bounded TCP connect (2 s timeout) to
  `target:port`. The **port cannot be derived from a DNS
  record**: `--health-port` sets it (default 443). The checker is
  **deliberately naive** (a successful connect ≠ healthy
  service) — improvable (HTTP, TLS, application-level) without
  format change.
- Entry TTL: `--health-ttl-secs` (default 5 s); expired entry =
  Unknown = available.

### Security

- GEOROUTE records are signed records like any other: verified
  against the chain (owner, sequence, record_hash) before any
  use. **No network trust without validation** — an unverified
  DHT GEOROUTE never answers.
- The client's position comes from the resolver's **local
  configuration** (`--geo-ip-table` or
  `--geo-country`/`--geo-continent`/`--geo-asn`/`--geo-region`),
  never from a network GeoIP API (no query leak, no dependency).
- The draw is deterministic and local: no secret, no
  unpredictability required (the security is that of the record
  signature, not of the draw).

### Complete example (reference spec)

`www.example.scone` publishes (apex `example.scone`, the whole
set serves every subname):

| Scope | Endpoint | Priority | Weight |
|---|---|---|---|
| US (840) | 10.1.0.1 | 10 | 70 |
| US (840) | 10.1.0.2 | 10 | 30 |
| US (840) | 10.2.0.1 | 20 | 100 |
| TW (158) | 10.3.0.1 | 10 | 80 |
| TW (158) | 10.4.0.1 | 10 | 20 |
| TW (158) | 10.2.0.1 | 20 | 100 |
| WORLD | 10.0.0.1 | 100 | 100 |

Expected behavior:

- **US** client → 10.1.0.1 (≈70%) or 10.1.0.2 (≈30%); if both
  are Unhealthy → failover p20 → **10.2.0.1**;
- **TW** client → 10.3.0.1 (≈80%) or 10.4.0.1 (≈20%);
- **FR** client (250) → no FR or EU route in the set → WORLD →
  **10.0.0.1**;
- everything Unhealthy → no geo decision → NODATA (NOERROR, 0
  answers);
- **A** question on an IPv6-only route → empty answer (never an
  IPv6 in A);
- without `--geo-dns` → same question, classic behavior (NODATA:
  the GEOROUTE has no wire A rdata).

### CLI options (`scone relay`)

| Option | Default | Role |
|---|---|---|
| `--geo-dns` | off | Arms the resolver's GeoDNS (absolute opt-in) |
| `--geo-ip-table <path>` | — | **Local GeoIP table** (see below). Takes PRECEDENCE over all static `--geo-*` options |
| `--geo-country <u16>` | — | Client country, ISO 3166-1 numeric (840=US…) — static |
| `--geo-continent <u16>` | — | Client continent, UN M49 (150=EU…); derived from country if omitted — static |
| `--geo-asn <u32>` | — | Client AS (64500…) — static, new in v2 |
| `--geo-region <u32>` | — | Client subdivision (opaque publisher identifier) — static, new in v2 |
| `--health-check` | off | Enables checks (health failover) |
| `--no-health-check` | off | **Explicitly disables** checks — WINS over `--health-check` (see precedences) |
| `--health-ttl-secs <u64>` | 5 | TTL of a health entry |
| `--health-port <u16>` | 443 | TCP port for checks (not derivable from a DNS record) |

#### Local GeoIP table (`--geo-ip-table`)

Local TEXT file of prefixes, read **once** at startup (100,000
rules max, `MAX_GEOIP_RULES`). One rule per line, `#` =
comment, blank lines ignored:

```text
# prefix/mask        country continent asn     region
10.0.0.0/8           840   3          .       .
2001:db8::/32        250   150        64500   11
192.0.2.0/24         158   9
```

- `prefix/mask`: IPv4 (`len ≤ 32`, converted to 16-byte form,
  zero tail — same convention as the wire) or IPv6 (`len ≤ 128`).
  **Host bits must be zero** (`10.1.2.3/8` is refused);
- `country`: **numeric** ISO 3166-1 `u16`, required;
- `continent`: UN M49 `u16`, optional (`.` or omitted = absent,
  the continent can then be derived from the country);
- `asn`: `u32`, optional (`.` = absent);
- `region`: opaque `u32`, optional (`.` = absent).

Resolution: the resolver's IP (today the reference IP — the
**per-client** one, source IP of the DNS query, is the natural
evolution, not yet wired) is compared in 16 bytes; the rule with
the **longest matching prefix** wins (on a tie, the first in
file order); no match → empty position (World).

**Strictness**: the file is operator-configured — an invalid
line (malformed prefix, non-numeric country, unknown continent,
excess field…) is a **startup error** carrying the **offending
line number**; never a silently ignored line, never a partial
table.

#### Option precedences

- **Locator**: `--geo-ip-table` **takes precedence** over the
  static options
  (`--geo-country`/`--geo-continent`/`--geo-asn`/`--geo-region`),
  which are ignored when the table is provided. Without a table,
  the static options apply.
- **Health**: `--no-health-check` is a **dedicated flag that
  wins** over `--health-check` — the combination
  `--health-check --no-health-check` leaves checks disabled, with
  no input-order ambiguity. Without `--health-check`, checks are
  off by default: **Unknown = available**, the geo answer goes
  out even if the target is dead. (A `NoopChecker` — always
  Healthy — also exists to mount the health structure without
  setting aside any endpoint: tests/paranoia.)

Example:

```bash
scone relay --dns 127.0.0.1:5353 \
            --geo-dns --geo-ip-table /etc/scone/geoip.txt \
            --no-health-check
```

Static v2 example:

```bash
scone relay --dns 127.0.0.1:5353 \
            --geo-dns --geo-country 250 --geo-asn 64500 --geo-region 11 \
            --health-check --health-port 443
```

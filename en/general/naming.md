# Naming

## Alphabet

Lowercase ASCII letters, digits, and hyphens (LDH, RFC 1123):
`[a-z0-9-]`, with the hyphen neither in first nor last position of a
label. No uppercase, no underscore, no unicode.

Validation is **strict**: a non-canonical name is rejected, never
corrected. `"ABC"` → error (no silent lowercasing). Rationale: a
protocol boundary must be deterministic; implicit canonicalization
creates identity ambiguities.

## TLD

A TLD is a DNS label like any other: the generic label limit
(RFC 1035: 63 bytes) applies, not a shorter TLD-specific limit.

- pattern: `[a-z0-9-]{1,63}` (LDH, no leading/trailing hyphen)
- coded limit: `TldName::MAX_LEN = 63` bytes
- valid: `uip`, `com`, `test`, `abc12`, `0x`, `a-b`, a TLD of
  exactly 63 bytes
- invalid: a 64-byte TLD, `ABC`, `-ab`, `ab-`, `éxemple`,
  empty string

## Domains

- shape: `label[.label]*.tld` — at least two labels (`name.tld`);
- each label: LDH `[a-z0-9-]{1,63}`, hyphen neither leading nor trailing;
- total length: ≤ 253 bytes;
- no empty labels (`a..b`, `.a`, `a.` invalid);
- the last label must be a valid TLD.

### Subdomains

The `DomainName` type accepts the full hierarchy from the start:

```text
example.uip
shop.example.uip
api.shop.example.uip
```

A subdomain has a `DomainId` **distinct** from its parent's.
Subdomain allocation rules (rights of the parent's owner, etc.) are
future work.

## Canonicalization

A validated name is already canonical: `canonical() == as_str()`.
Therefore rejected today: uppercase, punctuation other than the `.`
separator, spaces, unicode, invisible characters, etc.

## DomainId

```text
DomainId = BLAKE3-256("SCONE-DOMAIN-V1" || canonical_name)
```

- deterministic: same canonical name ⇒ same id, everywhere, forever;
- injective in practice: two distinct names ⇒ two distinct ids;
- the raw name must **never** serve as an internal identifier
  (blockchain keys, DHT keys) — only for input and display;
- the `SCONE-DOMAIN-V1` prefix isolates derivation domains: changing
  the rule (alphabet, normalization) amounts to creating a new
  versioned namespace (`SCONE-DOMAIN-V2`), without rewriting the
  existing one.

### Textual representation

**Lowercase** hexadecimal encoding (`0-9a-f`) of the 32 raw bytes:
exactly 64 characters, no prefix (`0x`), never truncated.

```text
DomainId([0xab; 32]) → "ababababababababababababababababababababababababababababababababab"
DomainId([0x00; 32]) → "0000000000000000000000000000000000000000000000000000000000000000"
```

This form is produced by `Display` (the Rust `Debug` prints
`DomainId(<hex>)`, same string). Any output consumed by a human or a
machine outside the canonical binary format — logs, error messages,
CLI (`scone show`) — MUST use this representation and no other (no
truncation, no prefix, no uppercase).

## Limits and future rules (not settled)

- internationalized names (IDN) via punycode;
- minimum length of second-level labels;
- expiration/renewal of domain claims.

## TLDId (M7a)

A registrable TLD carries its own identity, distinct from that of
any domain:

```text
TldId = BLAKE3-256("SCONE-TLD-V1" || tld)
```

- the `SCONE-TLD-V1` prefix is **distinct** from `SCONE-DOMAIN-V1`:
  the TLD and domain id spaces are disjoint by construction —
  registering the TLD `uip` (a claim on the `*.uip` namespace) can
  never squat a domain's identity, nor the reverse;
- same display rules as `DomainId`: lowercase hex, 64 characters,
  never truncated;
- pinned vector (derivation pinned by test):
  `TldId("uip") = ad5a86d68643d5c22d6a959bb1a315530c77dfda241f1ac1a8107450f3fab25e`.

The `RegisterTld` transaction (signed claim of a TLD, `owner`
always recomputed from the embedded key) is part of the
`Transaction` enum and the wire format since M7b: discriminant
`0x93` (arbitrary opaque constant), signed payload
`SCONE-TX-SIG-V1`, full rules in `/docs/en/technical/transactions.md`.
A domain's `RegisterDomain` meanwhile carries the canonical name in
cleartext.

## TLD reservation/governance

Opened by M7a: `RegisterTld` claims a free TLD for the benefit of
the identity derived from the signing key. Since M8b governance is
complete: claim with PoW (per-network difficulty), namespace
**closed** by default (assign-only), explicit opening
(`SetTldOpen`), transfer (`TransferTld`), abandonment
(`RevokeTld`, re-claimable). Domains expire after 1 year
(renewable, 30-day grace — see `/docs/en/technical/blockchain.md`).

**ICANN root TLDs**: reserved. `RegisterTld` on a TLD from the
pinned IANA list (`is_icann_tld`, ~1,440 entries + RFC 2606) is
rejected with a typed error (`IcannTldReserved`) — the legacy root
belongs to the existing DNS. At resolution time, these names are
forwarded to the configured upstreams (`--dns-upstream`) or
REFUSED without an upstream. TLDs added by ICANN later remain
claimable (pinned list: consistency > freshness).


## Lifecycle economics — who pays what, when

The authorization model follows the nature of the resource: work
(PoW) pays for what enters the commons, a signature (PoS) governs
what belongs to someone. Normative summary:

### Private TLD

- **Birth — PoW**: claimed by proof of work, difficulty by length
  (mainnet: 1 char ≈ 34 d of computation; floor ≈ 4.3 d at 10+
  chars). The claimant becomes the owner and holds a **PoS seat**
  (validator eligibility pool — heartbeats required to stay fresh,
  operator delegation available).
- **Life — PoS**: the owner's signature authorizes everything in
  the namespace: `assign` (grant a domain), `set-tld-open`
  (irreversible — **releases the seat**), `transfer`, `revoke`, and
  the renewal of the domains living under it.
- **Death — GC**: the TLD dies when its last domain expires
  (derived deadline, maintained O(1)). The seat dies with it.

### Open TLD (born open)

- **Birth — reduced PoW** (a third cheaper than private: a common
  resource nobody governs). **No PoS seat**.
- **Life**: nobody signs anything — domains register
  self-service. Its death clock follows its most recent domain's
  expiry (a renewal postpones it).
- **Death — GC** when no domain lives under it anymore.

### Domain under an open TLD

- **Birth — PoW** (reduced curve vs TLD: mainnet 1 char ≈ 8.5 d,
  floor ≈ half a day).
- **Renewal — PoW identical to the birth**: every term of life is
  paid in work, a domain cannot be "collected" for free (≤ 3 years
  in advance).
- **Death**: expiry → 30 d grace (renewable) → GC → the name
  becomes claimable by **anyone** with a fresh PoW.

### Domain under a private TLD

- **Birth — the TLD owner's PoS**: only the owner (or its delegated
  operator) grants (`assign`) — the signature is the authority, no
  PoW.
- **Renewal — the TLD owner's PoS**: signed by the TLD owner, empty
  `proof` field.
- **Death**: grace then GC; re-claiming goes through the owner's
  `assign` again.

### Cross-cutting invariants

- Every PoW is salted by **epoch**, with a difficulty-proportional
  **backward** validity window (a multi-week mine still lands even
  if the epoch turned).
- PoS signer equivocation is **slashed** (lifetime ban — M9).
- Every operation re-verifies everything at apply time: nothing is
  taken on faith (hashes, proofs and signatures recomputed).

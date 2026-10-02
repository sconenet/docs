# Scone documentation

The documentation is **bilingual** and **normative**: it must enable an
independent implementation without reading the Rust code. Any
doc/code divergence is a bug (identify the source of truth, fix it).

- [`fr/`](fr/README.md) — documentation en français
- [`en/`](en/README.md) — documentation in English

Each language tree mirrors the same structure:

- **`general/`** — how it works: protocol concepts and rules,
  readable without the binary-format details;
- **`technical/`** — developer specifications: exact binary formats,
  hash formulas, bounds, APIs and per-component behaviour.

The two trees are kept in sync: a change in one must be mirrored in
the other (same file names).

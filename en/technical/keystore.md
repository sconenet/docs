# Keystore — `.sconekey` keyfile format specification

> Normative document. Any divergence between this document and
> `crates/scone-keystore/src/lib.rs` is a bug (identify the source
> of truth, fix it). Implementation reference: crate
> `scone-keystore` (v2).

## Purpose

Store an Ed25519 private key (the *seed*, 32 bytes) on disk,
encrypted with a passphrase. The key **never** touches disk in
cleartext; all secrets (passphrase, derived material) are zeroized
in memory after use.

## Overview

- **KDF**: Argon2id (v0x13), default parameters `m_cost` = 64 MiB,
  `t_cost` = 3, `p_cost` = 1 (defaults of the `argon2` crate).
- **Encryption**: XChaCha20-Poly1305 (AEAD), 32-byte key derived
  from the KDF, 24-byte random nonce per file.
- **Encrypted content**: exactly the Ed25519 seed (32 bytes); the
  Poly1305 tag (16 bytes) is stored as a suffix of the ciphertext
  (standard output of the AEAD `encrypt` API).

## Byte sealing (`bytes` module, web)

`scone_keystore::bytes::{seal_to_bytes, open_from_bytes}` produces
and consumes the SAME format without touching the filesystem: the
web client generates the identity in the browser (`scone-crypto` +
bip39) and seals the exact keyfile bytes (downloadable, openable
by the CLI as-is — interoperability test in the suite). The native
path (`create`/`create_with_phrase`) delegates to the same
sealing: a single encoding of the format, three destinations.

## Binary format (v2) — 52-byte header + public metadata + ciphertext

```text
Offset  Size        Field
0       9           magic: "SCONEKEY1" (ASCII — unchanged since v1)
9       1           version: u8 = 2
10      1           m_cost_mib: u8 (m_cost in MiB; m_cost_kib = value × 1024)
11      1           t_cost: u8 (Argon2id iterations)
12      16          Argon2id salt (random per file)
28      24          XChaCha20-Poly1305 nonce (random per file)
52      32          public_key: Ed25519 public key (32 bytes, IN THE CLEAR)
84      8           created_at: u64 big-endian (Unix, seconds)
92      1           label_len: u8 (0..=255)
93      label_len   label: UTF-8 (readable without the passphrase, may be empty)
—       48          ciphertext (32 bytes encrypted seed) || Poly1305 tag (16 bytes)
```

The `public_key`/`created_at`/`label` metadata is public: readable
**without the passphrase** (`public_metadata_from_bytes`). The
public key is public by nature; the label is just a display name —
hiding it buys nothing and costs the no-passphrase listing. The
total length depends on the label: 141 bytes with an empty label
(KEYFILE_LEN + 32 + 8 + 1), plus the label's length.

**v1 (superseded, exactly 100 bytes)**: magic ‖ version=1 ‖ m ‖
t ‖ salt ‖ nonce ‖ ciphertext — the public key was NOT stored (a
locked file was anonymous). v1 files remain **readable**
(metadata `None`, passphrase required for `identity show`;
re-sealing via `scone identity restore` migrates them to v2).

### Design choices

- **KDF parameters in the file**: `m_cost_mib` and `t_cost` are
  read from the header (not hardcoded at read time), allowing them
  to be raised in future versions without breaking existing files.
- **Security bounds at read time**: a file declaring
  `m_cost_mib` > 64 or `t_cost` = 0 or > 10 is rejected
  (`Error::InvalidParams`) *before* any allocation — a malicious
  file cannot force a node to allocate gigabytes or loop
  indefinitely. The default at write time is `m_cost_mib` = 64,
  `t_cost` = 3.
- **Explicit magic + version**: a file that is not a keyfile
  (`Error::InvalidFile`) or of an unsupported version
  (`Error::UnsupportedVersion`) is rejected cleanly, without a
  panic.
- **Wrong passphrase**: the Poly1305 authentication failure does
  not indicate *why* it fails; a wrong passphrase and a corrupted
  ciphertext are therefore indistinguishable and surface as the
  same distinct error `Error::WrongPassphrase`.

## Encryption key derivation

```text
key = Argon2id(passphrase, salt, m_cost = m_cost_mib × 1024 KiB,
               t_cost = t_cost, p_cost = 1, output = 32 bytes)
```

Then:

```text
ciphertext || tag = XChaCha20-Poly1305-Seal(key, nonce, plaintext = seed)
```

No associated data (AAD) in v1. The 24-byte nonce, random per
file, makes nonce reuse risk negligible (keyfiles are written only
once).

## API (`scone-keystore`)

| Function | Role |
|---|---|
| `create(path, passphrase) -> GeneratedKey` | Generates a key, writes the encrypted keyfile (creates parent directories); **refuses to overwrite** an existing file (`Error::KeyfileExists`); returns the key in memory |
| `create_overwriting(path, passphrase) -> GeneratedKey` | Same, but explicitly replaces an existing keyfile |
| `open(path, passphrase) -> SigningKey` | Decrypts and returns the signing key |
| `list(dir) -> Vec<KeyEntry>` | Lists the `*.sconekey` files in a directory (sorted by name); missing directory = empty list |

`GeneratedKey.signing_key` is zeroized on drop by `ed25519-dalek`
(default `zeroize` feature). The derived AEAD key, the seed, and
the decrypted plaintext are wrapped in `zeroize::Zeroizing`:
Drop covers all exit paths, including errors propagated by `?`.

## File permissions

On Unix, the keyfile is created with permissions `0600`
(read/write by the owner only), via `OpenOptions` +
`set_permissions` — regardless of the process umask. Existence and
creation are atomic (`create_new`, `O_EXCL`): two concurrent
generations cannot silently overwrite each other.

## CLI

```bash
scone identity generate <n> [--dir <directory>] [--passphrase-env VAR] [--force]
scone identity list [--dir <directory>]
scone identity show <n> [--dir <directory>] [--passphrase-env VAR]
```

- Default directory: `$HOME/.scone/keys`.
- The identity name is strictly validated: empty, `.`, `..`, `/`,
  `\\`, NUL, leading dot, and the `.sconekey` suffix are rejected
  (the name becomes a file name inside the keys directory — no
  path component can escape the keystore).
- Without `--passphrase-env`, the passphrase is requested on the
  terminal **with masked input** (`rpassword`: no echo; twice with
  confirmation for `generate`).
- `identity generate` **displays the BIP39 recovery phrase (24
  words)** exactly once — fresh 256-bit entropy, the words alone
  regenerate the same key (`scone identity restore`); they are
  never stored anywhere.
- `identity restore` reads the phrase **never as a process
  argument** (visible via `ps` and shell history): masked
  prompt, `--phrase-file <file>` or `--phrase-env <VAR>`. It
  **refuses to overwrite** an existing keyfile by default
  (`--force` replaces it explicitly); the auto-generated anchor
  passphrase is written `0600` and a write failure is reported,
  never swallowed.
- `identity generate` **refuses to overwrite** an existing
  keyfile; `--force` explicitly replaces it (new salt/nonce, new
  key, new phrase).
- `identity list` shows, for v2 files, the public key and the
  label **without the passphrase** (public metadata).
- `identity show` reads the v2 keyfile's public metadata and
  displays the public key, the `OwnerId` in lowercase hex (64
  characters, same convention as `DomainId`), the label, and the
  creation date **without the passphrase**; a v1 file (no
  metadata) still asks for it.

## What this format deliberately does not do

- No multiple keys per file (one seed per file).
- No passphrase change (if it ever comes: rewrite the file with a
  new salt/nonce).


## Default identity (cli-simplify-v1)

Commands that need a signing identity (`domain register`,
`domain set`, `domain update`, `domain renew`) use the keystore's
SINGLE identity without `--identity` when there is exactly one.
With several identities the command fails and lists them
(`scone identity list`); pass `--identity <name>` explicitly.

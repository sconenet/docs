# Keystore — `.sconekey` keyfile format specification

> Normative document. Any divergence between this document and
> `crates/scone-keystore/src/lib.rs` is a bug (identify the source
> of truth, fix it). Implementation reference: crate
> `scone-keystore` (v1).

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

## Binary format (v1) — exactly 100 bytes

```text
Offset  Size        Field
0       9           magic: "SCONEKEY1" (ASCII)
9       1           version: u8 = 1
10      1           m_cost_mib: u8 (m_cost in MiB; m_cost_kib = value × 1024)
11      1           t_cost: u8 (Argon2id iterations)
12      16          Argon2id salt (random per file)
28      24          XChaCha20-Poly1305 nonce (random per file)
52      48          ciphertext (32 bytes encrypted seed) || Poly1305 tag (16 bytes)
```

All integers are raw unsigned bytes, with no endianness (single-
byte fields only). Total length: 100 bytes.

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
scone identity generate --name <n> [--dir <directory>] [--passphrase-env VAR] [--force]
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
- `identity generate` **refuses to overwrite** an existing
  keyfile; `--force` explicitly replaces it (new salt/nonce, new
  key).
- `identity show` displays the public key and the `OwnerId` in
  lowercase hex (64 characters, same convention as `DomainId`),
  recomputed via `scone-core` (`PublicKeyRef` → `OwnerId`) —
  never duplicated.

## What this format deliberately does not do

- No multiple keys per file (one seed per file).
- No passphrase change (if it ever comes: rewrite the file with a
  new salt/nonce).
- No metadata (creation date, comment): keep the disk footprint
  and attack surface minimal.

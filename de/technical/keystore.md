# Keystore — Spezifikation des `.sconekey`-Schlüsseldateiformats

> Normatives Dokument. Jede Abweichung zwischen diesem Dokument und
> `crates/scone-keystore/src/lib.rs` ist ein Bug (die Quelle der
> Wahrheit ermitteln und beheben).
> Implementationsreferenz: Crate `scone-keystore` (v2).

## Zweck

Einen privaten Ed25519-Schlüssel (den *Seed*, 32 Bytes) auf
der Festplatte speichern, verschlüsselt mit einer Passphrase. Der
Schlüssel berührt die Festplatte **nie** im Klartext; alle
Geheimnisse (Passphrase, abgeleitetes Material) werden im Speicher
nach Gebrauch genullt (zeroized).

## Überblick

- **KDF**: Argon2id (v0x13), Standardparameter `m_cost` = 64 MiB,
  `t_cost` = 3, `p_cost` = 1 (Standards des Crate `argon2`).
- **Verschlüsselung**: XChaCha20-Poly1305 (AEAD), 32-Byte-Schlüssel
  abgeleitet aus der KDF, 24 Bytes zufällige Nonce pro Datei.
- **Verschlüsselter Inhalt**: genau der Ed25519-Seed (32 Bytes); das
  Poly1305-Tag (16 Bytes) wird als Suffix des Geheimtexts
  gespeichert (Standardausgabe der AEAD-`encrypt`-API).

## Byte-Versiegelung (`bytes`-Modul, Web)

`scone_keystore::bytes::{seal_to_bytes, open_from_bytes}` erzeugt
und konsumiert DASSELBE Format, ohne das Dateisystem zu berühren:
der Web-Client erzeugt die Identität im Browser (`scone-crypto` +
bip39) und versiegelt die exakten Schlüsseldatei-Bytes
(herunterladbar, von der CLI unverändert öffenbar —
Interoperabilitätstest in der Suite). Der native Pfad
(`create`/`create_with_phrase`) delegiert an dieselbe Versiegelung:
eine einzige Kodierung des Formats, drei Ziele.

## Binärformat (v2) — 52-Byte-Header + öffentliche Metadaten + Geheimtext

```text
Offset  Größe        Feld
0       9           magic: "SCONEKEY1" (ASCII — unverändert seit v1)
9       1           version: u8 = 2
10      1           m_cost_mib: u8 (m_cost in MiB; m_cost_kib = Wert × 1024)
11      1           t_cost: u8 (Argon2id-Iterationen)
12      16          Argon2id-Salz (zufällig pro Datei)
28      24          XChaCha20-Poly1305-Nonce (zufällig pro Datei)
52      32          public_key: öffentlicher Ed25519-Schlüssel (32 Bytes, IM KLARTEXT)
84      8           created_at: u64 Big-Endian (Unix, Sekunden)
92      1           label_len: u8 (0..=255)
93      label_len   label: UTF-8 (ohne Passphrase lesbar, darf leer sein)
—       48          Geheimtext (32 Bytes verschlüsselter Seed) || Poly1305-Tag (16 Bytes)
```

Die Metadaten `public_key`/`created_at`/`label` sind öffentlich:
lesbar **ohne die Passphrase** (`public_metadata_from_bytes`). Der
öffentliche Schlüssel ist von Natur aus öffentlich; das Label ist
nur ein Anzeigename — es zu verbergen bringt nichts und kostet das
Listen ohne Passphrase. Die Gesamtlänge hängt vom Label ab: 141
Bytes mit leerem Label (KEYFILE_LEN + 32 + 8 + 1), plus der Länge
des Labels.

**v1 (abgelöst, genau 100 Bytes)**: magic ‖ version=1 ‖ m ‖ t ‖
Salz ‖ Nonce ‖ Geheimtext — der öffentliche Schlüssel wurde NICHT
gespeichert (eine gesperrte Datei war anonym). v1-Dateien bleiben
**lesbar** (Metadaten `None`, Passphrase erforderlich für
`identity show`; erneutes Versiegeln über `scone identity restore`
migriert sie zu v2).

### Entwurfsentscheidungen

- **KDF-Parameter in der Datei**: `m_cost_mib` und `t_cost` werden
  aus dem Header gelesen (beim Lesen nicht hartkodiert), sodass sie
  in künftigen Versionen erhöht werden können, ohne bestehende
  Dateien zu brechen.
- **Sicherheitsschranken zur Lesezeit**: eine Datei, die
  `m_cost_mib` > 64 oder `t_cost` = 0 oder > 10 deklariert, wird
  abgelehnt (`Error::InvalidParams`) *vor* jeder Allokation — eine
  bösartige Datei kann einen Knoten nicht zwingen, Gigabytes zu
  allokieren oder endlos zu schleifen. Der Standard zur Schreibzeit
  ist `m_cost_mib` = 64, `t_cost` = 3.
- **Explizites Magic + Version**: eine Datei, die keine
  Schlüsseldatei ist (`Error::InvalidFile`), oder eine nicht
  unterstützte Version (`Error::UnsupportedVersion`) wird sauber
  abgelehnt, ohne Panik.
- **Falsche Passphrase**: das Poly1300-Authentifizierungsversagen
  gibt nicht an, *warum* es scheitert; eine falsche Passphrase und
  ein korrumpierter Geheimtext sind daher ununterscheidbar und
  erscheinen als derselbe eigene Fehler `Error::WrongPassphrase`.

## Ableitung des Verschlüsselungsschlüssels

```text
key = Argon2id(passphrase, salt, m_cost = m_cost_mib × 1024 KiB,
               t_cost = t_cost, p_cost = 1, Ausgabe = 32 Bytes)
```

Dann:

```text
ciphertext || tag = XChaCha20-Poly1305-Seal(key, nonce, plaintext = seed)
```

Keine zugeordneten Daten (AAD) in v1. Die 24-Byte-Nonce, zufällig
pro Datei, macht das Nonce-Wiederverwendungsrisiko vernachlässigbar
(Schlüsseldateien werden nur einmal geschrieben).

## API (`scone-keystore`)

| Funktion | Rolle |
|---|---|
| `create(path, passphrase) -> GeneratedKey` | Erzeugt einen Schlüssel, schreibt die verschlüsselte Schlüsseldatei (erstellt Elternverzeichnisse); **weigert sich zu überschreiben** einer bestehenden Datei (`Error::KeyfileExists`); gibt den Schlüssel im Speicher zurück |
| `create_overwriting(path, passphrase) -> GeneratedKey` | Gleich, ersetzt aber explizit eine bestehende Schlüsseldatei |
| `open(path, passphrase) -> SigningKey` | Entschlüsselt und gibt den Signaturschlüssel zurück |
| `list(dir) -> Vec<KeyEntry>` | Listet die `*.sconekey`-Dateien in einem Verzeichnis (nach Name sortiert); fehlendes Verzeichnis = leere Liste |

`GeneratedKey.signing_key` wird beim Drop von `ed25519-dalek`
genullt (Standard-Feature `zeroize`). Der abgeleitete AEAD-Schlüssel,
der Seed und das entschlüsselte Klartext sind in
`zeroize::Zeroizing` eingewickelt: Drop deckt alle Ausgabepfade ab,
einschließlich der über `?` weitergegebenen Fehler.

## Dateiberechtigungen

Auf Unix wird die Schlüsseldatei mit Berechtigungen `0600` erstellt
(nur für den Eigentümer les-/schreibbar), über `OpenOptions` +
`set_permissions` — unabhängig vom Prozess-Umask. Existenz und
Erstellung sind atomar (`create_new`, `O_EXCL`): zwei gleichzeitige
Generierungen können sich nicht stillschweigend überschreiben.

## CLI

```bash
scone identity generate <n> [--dir <verzeichnis>] [--passphrase-env VAR] [--force]
scone identity list [--dir <verzeichnis>]
scone identity show <n> [--dir <verzeichnis>] [--passphrase-env VAR]
```

- Standardverzeichnis: `$HOME/.scone/keys`.
- Der Identitätsname wird strikt validiert: leer, `.`, `..`, `/`,
  `\\`, NUL, führender Punkt und das Suffix `.sconekey` werden
  abgelehnt (der Name wird ein Dateiname innerhalb des
  Keystore-Verzeichnisses — keine Pfadkomponente kann aus dem
  Keystore ausbrechen).
- Ohne `--passphrase-env` wird die Passphrase im Terminal
  **mit maskierter Eingabe** abgefragt (`rpassword`: kein Echo;
  zweimal mit Bestätigung bei `generate`).
- `identity generate` **zeigt die BIP39-Wiederherstellungsphrase (24
  Wörter)** genau einmal an — frische 256-Bit-Entropie; allein die
  Wörter regenerieren denselben Schlüssel (`scone identity
  restore`); sie werden nirgends gespeichert.
- `identity generate` **weigert sich zu überschreiben** einer
  bestehenden Schlüsseldatei; `--force` ersetzt sie explizit (neues
  Salz/Nonce, neuer Schlüssel, neue Phrase).
- `identity list` zeigt für v2-Dateien den öffentlichen Schlüssel
  und das Label **ohne die Passphrase** (öffentliche Metadaten).
- `identity show` liest die öffentlichen Metadaten der v2-Schlüsseldatei
  und zeigt den öffentlichen Schlüssel, die `OwnerId` in
  Hex-Kleinbuchstaben (64 Zeichen, dieselbe Konvention wie
  `DomainId`), das Label und das Erstellungsdatum **ohne die
  Passphrase**; eine v1-Datei (ohne Metadaten) fragt weiterhin
  danach.

## Was dieses Format bewusst nicht tut

- Keine mehreren Schlüssel pro Datei (ein Seed pro Datei).
- Kein Passphrasenwechsel (falls er je kommt: die Datei mit neuem
  Salz/Nonce neu schreiben).


## Standard-Identität (cli-simplify-v1)

Befehle, die eine Signieridentität benötigen (`domain register`,
`domain set`, `domain update`, `domain renew`), verwenden die
EINZIGE Identität des Keystore ohne `--identity`, wenn genau eine
vorhanden ist. Bei mehreren Identitäten scheitert der Befehl und
listet sie auf (`scone identity list`); `--identity <name>`
explizit übergeben.

# Protokoll

Referenzdokument für das Scone-Protokoll: Binärformat, kanonische
Kodierung, Transaktionen, DNS-Einträge, Blöcke, P2P-Nachrichten,
Grenzen und Versionskompatibilität. Die Referenzimplementierung ist
das Crate [`scone-protocol`](../../../crates/scone-protocol).

Mit **(vorläufig)** markierte Punkte sind nicht endgültig.

## Protokollversion

- Konstante: `PROTOCOL_VERSION: u32 = 1` (`scone-protocol`);
- getragen von `Hello.version` (Handshake) und `BlockHeader.version`;
- Kompatibilitätsregeln:
  - empfangene Version **höher** als die lokale Version → Zurückweisung
    (`ProtocolError::UnsupportedVersion`); der Knoten rät nicht das
    Format einer Version, die er nicht kennt;
  - Version `0` → ungültig;
  - es gibt nur einen Satz von Formaten: die
    Transaktions-Versions-Bytes (`0x01`) und die Blockversion müssen
    mit den lokalen Werten **exakt** übereinstimmen;
- keine Major/Minor-Aufteilung in dieser Phase: eine einzige ganze
  Zahl, bei jedem Wire-Format-Bruch erhöht.

## Grundsätze des Binärformats

- binär, nie JSON für Protokolldaten;
- deterministisch und kanonisch: zwei logisch gleiche Objekte
  erzeugen **exakt dieselben Bytes**;
- sequenzielles Parsen, wo möglich ohne Allokation (Byte-Borrows);
- alle Netzwerkeingaben sind unvertrauenswürdig: Längen und Zähler
  werden **vor** jeder Allokation geprüft; Dekodieren gerät nie in
  Panik.

### Byte-Reihenfolge (Endianness)

- alle Ganzzahlen sind LEB128-Varints (7-Bit-Gruppen, niederwertigste
  zuerst): native Byte-Reihenfolge findet keine Anwendung;
- IPv4/IPv6-Adressen und 32-Byte-Bezeichner: rohe Byte-Arrays, ohne
  Längenpräfix;
- es gibt keine Ganzzahl fester Breite im Format: eine einzige
  Kodierregel für ganze Zahlen.

### Varint (minimales LEB128)

Jedes Byte trägt 7 Datenbits; das höchste Bit ist bei jedem Byte 1,
außer beim letzten. Eine `u64` passt in 1 bis 10 Bytes.

| Wert | Bytes (hex) |
|---|---|
| 0 | `00` |
| 1 | `01` |
| 127 | `7f` |
| 128 | `80 01` |
| 255 | `ff 01` |
| 16.384 | `80 80 01` |
| 2^32 | `80 80 80 80 10` |
| u64::MAX | `ff ff ff ff ff ff ff ff ff 01` |

Die Kodierung muss **minimal** sein: überdimensionierte Formen werden
beim Dekodieren abgelehnt (`InvalidVarint`). Das ist die notwendige
Bedingung dafür, dass ein Wert nur genau eine gültige Bytefolge hat
(kanonische Kodierung).
Zurückweisungen: `80 00` (nicht minimal), `ff×9 02` (u64-Überlauf),
`ff×10` (zu lang).

### Zeichenketten und Bytes

- `bytes`: Varint-Länge + rohe Bytes;
- `str`: ebenso, mit obligatorischer UTF-8-Dekodierung;
- jede Länge ist begrenzt (siehe [Grenzen](#grenzen)) und wird vor
  dem Lesen geprüft.

### Bezeichner

| Typ | Wire-Darstellung |
|---|---|
| `DomainId` | 32 rohe Bytes |
| `TldId` | 32 rohe Bytes |
| `OwnerId` | 32 rohe Bytes |
| `PublicKeyRef` | 32 rohe Bytes |
| `RecordHash` | 32 rohe Bytes |
| `BlockHash` | 32 rohe Bytes |
| `MerkleRoot` | 32 rohe Bytes |
| `DomainName` | `str` (≤ 253 Bytes, von `scone-core` revalidiert) |

Ableitungen (`BLAKE3-256` + `SCONE-DOMAIN-V1`-Präfixe usw.) bleiben
durch `scone-core` definiert (siehe `/docs/en/general/naming.md`).

## Transaktionen

```text
Transaction = disc u8 || version u8 || payload
```

| Diskriminant | Typ |
|---|---|
| `0x21` | `RegisterDomain` |
| `0x2A` | `RegisterTldPow` |
| `0x52` | `UpdateDomain` |
| `0x57` | `RenewTldPow` |
| `0x84` | `RegisterDomainPow` |
| `0x93` | `RegisterTld` |
| `0xC3` | `RenewDomainPow` |

Jeder Diskriminant ist eine beliebige opake Konstante (keine
logische Sequenz; `0x00` nie gültig) — vollständige normative
Tabelle in `/docs/en/technical/transactions.md`.

Auf den Diskriminanten folgt das Byte `version` (aktueller Wert
`0x01`): jeder andere Wert wird ausdrücklich abgelehnt
(`UnsupportedVersion(value)`). Die meisten Transaktionen sind
signiert (Ed25519) — AUSSER den vier signaturlosen
pow-to-owner-Formen (`0x2A`, `0x57`, `0x84`, `0xC3`): der
geminte Beweis IST die Autorisierung, und seine Challenge bindet
den Schlüssel des Begünstigten und das Netzwerk. Vollständige
Spezifikation, signierte Nutzlast und
Validierungsregeln in **`/docs/en/technical/transactions.md`**
(normativ).

Seit der Wire-Änderung 2026-09 **tragen Transaktionen kein Feld
`network` mehr**: die `network_id` (`str` ≤ 16) wird in die
**Signaturnutzlast** (signierte Formen) oder die **PoW-Challenge**
(pow-to-owner-Formen) injiziert — eine für ein Netzwerk gemachte
Tx geht nie auf ein anderes über (siehe
`/docs/en/technical/blockchain.md`).

### REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

`RegisterDomain` trägt den **kanonischen Namen im Klartext**, gefolgt
von seiner `DomainId` — die Konsistenz `domain_id == DomainId(name)`
wird beim Dekodieren erneut geprüft (neu berechnet, nie geglaubt).
`proof` trägt das Registrierungs-PoW (20 Bytes: Nonce + Epoche +
Schwierigkeit — Epochen-Salz `epoch = height/144` des anwendenden
Blocks, bei der Anwendung STRKT gegen die Netzwerkkonstante und die
Epoche des Blocks verifiziert; typisierte Zurückweisung
`ProofEpochMismatch` bei Abweichung — M8b, siehe
`transactions.md`). `public_key` ist der Ed25519-Schlüssel des
Signierenden (genau 32 Bytes); `owner` muss die BLAKE3-Ableitung
dieses Schlüssels sein (neu berechnet, nie geglaubt); `signature`
ist genau 64 Bytes.

### REGISTER_TLD (0x93)

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

Claim einer TLD (von der Domain-Registry getrenntes Register, M7):
die TLD wird **im Klartext getragen** wie der Name einer Domain (M8b
— die PoW-Challenge leitet sich vom Namen ab), und `tld_id` ist die
`SCONE-TLD-V1`-Ableitung dieses Namens, konstruktiv disjunkt von
jeder `DomainId`. Zustandsregeln (M8b): TLD frei + nicht
ICANN-reserviert (`IcannTldReserved` — ICANN-Root-TLDs sind von
niemandem beanspruchbar) + PoW mit der **längenbasierten**
Schwierigkeit des Labels (`pow_length`-Kurve — siehe
`blockchain.md`); beanspruchte ist geschlossen (nur assign).

### UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

Beim Dekodieren erneut geprüfte Invariante: `sequence > 0` und
`owner` == Ableitung von `public_key`. Kein `timestamp`: die
Reihenfolge läuft über `sequence`. Die Transaktion trägt nur
kompakte Referenzen — der vollständige DNS-Inhalt bleibt im DHT
(siehe unten).

## DNS-Einträge (DHT-Seite)

Vollständige DNS-Einträge gelangen nie in die Blockchain: sie werden
über den DHT ausgetauscht (`GetRecord`/`Record`-Nachrichten) und
on-chain nur durch ihren `RecordHash` committet.

### RecordData

```text
RecordData = type.v || ttl.v || payload(type)
```

Die Typ-Codes sind die **echten DNS-Codes** (IANA). Die TTL
(Sekunden, Varint) folgt dem Typ-Code und deckt den ganzen Eintrag
ab: Grenzen `MIN_TTL = 1` .. `MAX_TTL = 2.592.000` (30 Tage;
Standard `DEFAULT_TTL = 86.400`).

| Code | Typ | Nutzlast |
|---|---|---|
| 1 | A | 4 Bytes |
| 2 | NS | `name` |
| 5 | CNAME | `name` |
| 12 | PTR | `name` |
| 15 | MX | `preference.v`, `name` |
| 16 | TXT | `str` ≤ 4096 |
| 28 | AAAA | 16 Bytes |
| 33 | SRV | `priority.v`, `weight.v`, `port.v`, `target(name)` |
| 257 | CAA | `flags u8`, `tag` (1–15 B), `value` (`str` ≤ 4096) |
| 32768 | GEOROUTE | GeoRoute (siehe unten) |
| andere | Unknown | code.v + `bytes` ≤ 4096 |

Beispiel: `A 192.0.2.1` (TTL 86.400) → `01 80 a3 05 c0 00 02 01`
(Typ, TTL-Varint, Adresse).

Ein `Unknown` mit bekanntem Code wird beim Kodieren abgelehnt (beim
Dekodieren kann nie eines entstehen).

#### GEOROUTE (GeoDNS)

Privater Scone-Typ (nicht-IANA-Bereich). Eine geografische
Routing-Regel:

```text
GeoRoute  = scope || count.v(≤ 32) || endpoint…
scope     = 0x00                    -- Welt
          | 0x01 continent.v        -- UN M49
          | 0x02 country.v          -- ISO 3166-1 numerisch
          | 0x03 region.v           -- Regionscode
          | 0x04 asn.v              -- AS-Nummer
          | 0x05 prefix.addr[16] || prefix.len u8   -- IP-Präfix (normalisiert)
endpoint  = target || priority.v || weight.v
target    = 0x00 octets[4]          -- binäres IPv4
          | 0x01 octets[16]         -- binäres IPv6
```

**Kanonische Regel**: die Endpunkte eines gegebenen Scopes werden
**sortiert** kodiert (Priorität aufsteigend, dann Gewicht absteigend,
dann Ziel aufsteigend); das Sortieren erfolgt beim Kodieren, und ein
unsortiertes Wire wird beim Dekodieren abgelehnt (`NonCanonical`),
wie bei Eintragssätzen. Vollständige Endpunkt-Duplikate werden
abgelehnt.

**Validierung**: `priority > 0`, `weight ≤ 100` (0 = inaktiver
Endpunkt), ≤ 32 Endpunkte pro Scope (`GEO_MAX_ENDPOINTS`). Ein der
lokalen Tabelle unbekannter Länder-/Kontinentscode
(`continent_of_country`) wird auf Wire-Ebene **akzeptiert**: die
Tabelle des Resolvers kann aktueller sein als die des Veröffentlichers.
Die TTL ist die des übergeordneten `RecordData` (unverändert).

### DnsRecord

```text
domain_id[32] sequence.v expiration.v count.v(≤ 256) records…
```

**Kanonische Regel**: ein Eintragssatz ist eine ungeordnete
Menge. Er wird **sortiert nach lexikografischer Ordnung der
Einzeldarstellung jedes Eintrags** kodiert, streng steigend
(Duplikate abgelehnt). Ein unsortiertes Wire wird beim Dekodieren
abgelehnt (`NonCanonical`). Die Reihenfolge des ursprünglichen `Vec`
wird nicht transportiert.

### SignedDnsRecord

```text
DnsRecord owner[32] signature(bytes ≤ 128)
```

### RecordHash

```text
RecordHash = BLAKE3-256("SCONE-RECORD-V1" || canonical(DnsRecord))
```

Der Hash deckt **nur den Eintragsinhalt** ab; die Signatur des Owners
wird separat über dieselben kanonischen Bytes verifiziert. *
(Entscheidung: das Commitment deckt die Signatur nicht ab — eine
frühere Version dieses Dokuments erwähnte das „vollständige
SignedDnsRecord“; die Kopplung des Hashes an die Signatur brachte
nichts und band das Commitment an das Signaturverfahren.)*

## Blöcke

### BlockHeader

```text
version.v height.v prev_hash[32] tx_root[32] timestamp.v consensus(bytes ≤ 65.536)
```

- `version`: Protokollversion (Regeln oben);
- `height`: Höhe (Genesis = 0);
- `prev_hash`: Hash des vorherigen Blocks (Nullen für den
  Genesis-Block);
- `tx_root`: Commitment über die Transaktionsliste (Merkle-Baum,
  definiert von `scone-blockchain`, siehe
  `/docs/en/technical/blockchain.md`);
- `timestamp`: Reihenfolgewert (Unix, Sekunden);
- `consensus`: Nutzlast des Produzenten (`SCONE-BLOCK-V1 ‖ pk[32] ‖
  sig[64]`, siehe `blockchain.md`) — die Genesis eines Netzwerks
  trägt die rohe `network_id`, und eine Allokations-Genesis
  (Mainnet-Kickstarter) kann ein `ALLOC-V1`-Manifest (≤ 60 KiB)
  tragen; begrenzt durch `MAX_CONSENSUS_LEN` (64 KiB).

### Block

```text
BlockHeader tx_count.v(≤ 4096) transactions…
```

Sequenzielles Parsen; der Zähler wird **vor** der Allokation
begrenzt; jede Abschneidung wird abgelehnt. Anders als bei
Eintragssätzen ist die Transaktionsreihenfolge **bedeutsam**
(Konsensreihenfolge) und wird exakt bewahrt.

`BlockHash` (Hash des kanonischen Headers) und `tx_root` werden von
`scone-blockchain` über die hier definierten Kodierungen berechnet
(exakte Formeln in `/docs/en/technical/blockchain.md`).

## P2P-Nachrichten

```text
Message = disc u8 || payload
```

Transport (Framing, libp2p…) ist nicht Gegenstand; ein
Transport-Frame sollte `MAX_MESSAGE_LEN` (1 MiB) nicht
überschreiten.

| Disc | Nachricht | Richtung | Nutzlast | Grenze | Erwartete Antwort |
|---|---|---|---|---|---|
| `0x01` | `Hello` | beide | `version.v` | — | `Hello` |
| `0x02` | `Ping` | beide | — | — | `Pong` |
| `0x03` | `Pong` | beide | — | — | — |
| `0x04` | `GetBlock` | beide | `hash[32]` | — | `Block` |
| `0x05` | `GetBlocks` | beide | `start.v` `max.v` | `max ≤ 128` | ≤ `max` × `Block` |
| `0x06` | `Block` | beide | `Block` | tx ≤ 4096 | — |
| `0x07` | `Transaction` | beide | `Transaction` | — | — |
| `0x08` | `GetRecord` | beide | `domain_id[32]` | — | `Record` |
| `0x09` | `Record` | beide | `SignedDnsRecord` | — | — |
| `0x0a` | `Checkpoint` | beide | `Checkpoint` | Signer ≤ 64 | — |
| `0x0b` | `GetCheckpoints` | beide | — | — | Checkpoint-Fenster |
| `0x0c` | `GetStatus` | beide | — | — | `NodeStatus` |
| `0x0d` | `NodeStatus` | beide | Knotenzustand (Netzwerk, Tip, Höhe, NAT, Domains, Mempool, Peers, Finalität, frische Betreiber) | — | — |
| `0x0e` | `LookupName` | beide | `name(str)` | — | `NameState` |
| `0x0f` | `NameState` | beide | On-Chain-Zustand des Namens + TLD, inkl. der CAS-Generation (Mutationszähler, registered_at) | — | — |
| `0x10` | `GetOwnedNames` | beide | `owner[32]` | — | `OwnedNames` |
| `0x11` | `OwnedNames` | beide | lebende Domains + TLDs des Owners | — | — |
| `0x12` | `Error` | Antwort | Fehlermeldung auf Anwendungsebene (`str`) | — | — |

### Checkpoint (M5)

Ein aggregierter Checkpoint geht als Folgendes auf das Wire: Tag
`SCONE-CKPT-V1` || Rumpf `CheckpointData` (116 B, LE — identisch mit
dem Ende der Signatur-Preimage: `epoch(8) height(8) block_hash(32)
prev_checkpoint_hash(32) state_root(32) recovery(4)`) || `count.v` ||
`pk[32] ‖ sig[64]` pro Signer, **sortiert nach aufsteigendem pk, ohne
Duplikate**, `count ≤ 64` (`MAX_CHECKPOINT_SIGNERS`). Strenges
Dekodieren: Abschneidung, falscher Tag, nicht-kanonischer pk,
Duplikate oder unsortiert = Zurückweisung (`None`), nie Panik
(Korruption gefuzztest).

`GetCheckpoints` fordert das Finalitätsfenster an; die Antwort ist
das Fenster `checkpoint_window()` (höchstens `CHECKPOINT_KEEP`
Checkpoints, vom ältesten zum neuesten) über eine oder mehrere
`Checkpoint`-Nachrichten.

`TxId` ist jetzt durch `scone-blockchain` definiert
(`BLAKE3-256("SCONE-TX-V1" || canonical(Transaction))`, siehe
`/docs/en/technical/blockchain.md`). Mempool und fensterbasierte
Anti-Replay (`TxReplay`, `REPLAY_WINDOW_BLOCKS = 256`) sind im Relay
ausgeliefert (M4); nur `GetTransaction` bleibt zurückgestellt (seine
Definition hängt von den übrigen Konsensentscheidungen ab).

## Aufteilung Blockchain / DHT

```text
BLOCKCHAIN                      DHT
──────────                      ───
domain_id                       domain_id
owner                           SignedDnsRecord (vollständig, signiert)
sequence
record_hash
```

Die Chain sagt, *wem* etwas gehört und *was der aktuelle Zustand ist*
(sequence + record_hash). Der DHT liefert *den Inhalt*. Ein
DHT-Eintrag wird genau dann akzeptiert, wenn:

1. seine Signatur für den auf der Chain registrierten Owner gültig
   ist;
2. seine `sequence` mit der der Chain übereinstimmt (oder höher
   ist);
3. der über seine kanonische Kodierung neu berechnete `RecordHash`
   mit dem `record_hash` der Chain übereinstimmt.

## Grenzen

| Konstante | Wert | Rolle |
|---|---|---|
| `MAX_NAME_LEN` | 253 | Länge eines Namens |
| `MAX_TLD_LEN` | 63 | Länge einer TLD |
| `MAX_TXT_LEN` | 4096 | TXT-Zeichenkette |
| `MAX_PROOF_LEN` | 256 | Registrierungs-Proof |
| `MAX_SIGNATURE_LEN` | 128 | Signatur (Ed25519 = 64) |
| `MAX_RECORDS_PER_SET` | 256 | Einträge pro Satz |
| `MAX_DHT_RECORD_LEN` | 64 KiB | kodierter signierter DNS-Eintrag (DHT-Nutzlast: Wire, Speicherung, Gossip) — Gesamtschranke beim Kodieren UND vor jedem Dekodieren geprüft; 15–30× über jedem realistischen Satz (SPF+DKIM+DMARC ≈ 2–5 KiB), 16× unter der Transportgrenze. |
| `GEO_MAX_ENDPOINTS` | 32 | Endpunkte pro GeoRoute-Scope |
| `MAX_UNKNOWN_DATA` | 4096 | Daten eines unbekannten Typs |
| `MAX_TXS_PER_BLOCK` | 4096 | Transaktionen pro Block |
| `MAX_CONSENSUS_LEN` | 64 KiB | Konsensnutzlast eines Headers (erweitert für das `ALLOC-V1`-Genesis-Manifest — Mainnet-Kickstarter) |
| `MAX_BLOCKS_PER_REQUEST` | 128 | Blöcke pro `GetBlocks` |
| `MAX_MESSAGE_LEN` | 1 MiB | Transport-Frame (empfohlen) |

Jede Überschreitung → `LimitExceeded`, vor der Allokation erkannt: ein
bösartiger Peer kann beim Dekodieren keine unverhältnismäßige
Allokation erzwingen.

## Kanonische Kodierung

Grundregel:

```text
kanonisches Objekt → kanonische Binärkodierung → Hash / Signatur
```

- nur minimale Varints;
- Eintragssätze streng steigend sortiert;
- ein logisch gleiches Objekt hat genau **eine** gültige Kodierung;
- das Dekodieren lehnt nicht-kanonische Formen ab (`NonCanonical`),
  also gibt es keine mehreren gültigen Kodierungen für dasselbe
  Objekt.

## Fehlerbehandlung

`ProtocolError` (`scone-protocol::error`):

| Variante | Bedeutung |
|---|---|
| `Truncated` | Eingabe endete mitten in einem Wert |
| `InvalidVarint` | nicht-minimaler, überlaufender oder zu langer Varint |
| `IntegerOutOfRange` | Ganzzahl passt nicht in ihren Zieltyp |
| `UnsupportedVersion` | Version null oder höher als lokal |
| `UnknownDiscriminant` | unbekannter Transaktions-/Nachrichten-Diskriminant |
| `LimitExceeded` | Länge oder Anzahl jenseits einer Grenze |
| `InvalidUtf8` | nicht-UTF-8-Zeichenkette |
| `TrailingBytes` | Bytes, die nach einem vollständigen Wert bleiben |
| `Validation` | Verletzung einer `scone-core`-Invarianten |
| `NonCanonical` | feldweise gültige, aber nicht-kanonische Kodierung |

Das Dekodieren gerät bei Netzwerkdaten nie in Panik.

## Signaturen

- Verfahren: **Ed25519** (RFC 8032) über `scone-crypto`
  (`ed25519-dalek`, Verifikation `verify_strict` — maleable
  Signaturen und schwache Schlüssel werden abgelehnt);
- **Chain-Transaktionen**: jede `RegisterDomain` / `RegisterTld` /
  `UpdateDomain` bettet `public_key` (32 B) + `signature` (64 B) über
  der Nutzlast `"SCONE-TX-SIG-V1" || canonical_encode(tx_without_signature)`
  ein; vollständige normative Spezifikation:
  `/docs/en/technical/transactions.md`;
- **DHT-Einträge** (`SignedDnsRecord`): die signierte Nutzlast ist
  die kanonische Kodierung des `DnsRecord`; die Verifikation kreuzt
  `record + owner + signature` mit der in der Blockchain
  registrierten Identität;
- Querschnittsregel: `owner` und alle Hashes/Commitments werden
  **neu berechnet**, nie aus einem gelieferten Wert übernommen.

## Konsensregeln

Ausgeliefert (beschrieben in `/docs/en/technical/blockchain.md`):
signierter Blockproduzent (`SCONE-BLOCK-V1`-Nutzlast, per
Checkpoint gewähltes Komitee), Finalität durch aggregierte
Checkpoints (`SCONE-CKPT-V1`, Fork-Wahl an der
Finalitätsuntergrenze), Registrierungs-PoW nach Namenslänge.
Offene Themen: feinkörnige Anti-Spam-Gebühren, Rotation des
Owner-Schlüssels.

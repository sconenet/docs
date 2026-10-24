# Signierte Transaktionen (Format v1)

**Normatives** Referenzdokument für das signierte Transaktionsformat.
Es spezifiziert die Felder, die exakte Wire-Reihenfolge, die
Größen, die signierte Nutzlast und die Validierungsregeln, um eine
unabhängige Implementierung ohne Lektüre des Rust-Codes zu
ermöglichen. Die Referenzimplementierung ist
[`scone-protocol`](../../../crates/scone-protocol) (Kodierung) und
[`scone-blockchain`](../../../crates/scone-blockchain) (Validierung).

`PROTOCOL_VERSION` = **1**. Es gibt nur einen Satz von Formaten: Jedes
Versionsbyte außer `0x01` (in einer Transaktion) oder abweichend von
der lokalen Version (in einem Block) wird mit `UnsupportedVersion`
abgelehnt.

## Überblick

```text
Transaction = disc u8 || version u8 || payload
```

| Feld | Größe | Rolle |
|---|---|---|
| `disc` | 1 B | Transaktionstyp (Tabelle unten) |
| `version` | 1 B | **genau `0x01`** (Tx-Formatversion) |

**Transaktionen tragen kein `network`-Feld mehr auf dem Wire**
(Änderung 2026-09): die `network_id` (`str` ≤ 16, ASCII `[a-z0-9-]` —
z. B. `scone-testnet`, 13 Bytes) lebt **nur in der
Signaturnutzlast** — sie wird dort vom Signer als erstes Feld
injiziert (siehe [Signierte Nutzlast](#signierte-nutzlast)) — und
bleibt pro Netzwerk im PoW-Digest. Eine für ein Netzwerk
**signierte** Transaktion verifiziert daher nie auf einem anderen:
ihre Signatur scheitert (`InvalidSignature`) gegen die von der Chain
neu berechnete Nutzlast, die ihr eigenes Netzwerk injiziert (siehe
`/docs/en/technical/blockchain.md` — Netzwerke und Genesis).

### Diskriminanten (normative Tabelle)

Jeder Diskriminant ist eine **feste, beliebige opake Konstante**: Die
Werte sind absichtlich verschieden gewählt, damit das Wire **nie**
eine logische Ordnung zwischen Typen suggeriert (keine
0x01/0x02/0x03…-Sequenz). `0x00` ist **nie** ein gültiger Typ
(typisches Korruptionsbyte). Künftige Typen nehmen jeden freien
Wert, einmalig in dieser Tabelle dokumentiert.

| Diskriminant | Typ | Rolle |
|---|---|---|
| `0x21` | `RegisterDomain` | Claim einer Domain |
| `0x52` | `UpdateDomain` | neue Version der DNS-Daten einer Domain |
| `0x93` | `RegisterTld` | Claim einer Top-Level-Domain (M7) |
| `0x47` | `TransferTld` | Übertragung einer TLD an einen neuen Owner (M8a) |
| `0x6B` | `RevokeTld` | Aufgabe einer TLD (M8a) |
| `0xB8` | `SetTldOpen` | **unumkehrbare** Öffnung einer TLD zur Selbstregistrierung (einseitig, Owner-Option; `open=false` ist eine typisierte Zurückweisung) |
| `0xD4` | `AssignDomain` | direkte Zuweisung einer Domain durch den Eigentümer der TLD (M8a) |
| `0x3C` | `RenewDomain` | Verlängerung der Registrierung einer Domain (M8a; trägt den kanonischen Namen seit Owner-Option — eine offene TLD folgt dem Ablauf) |
| `0x8F` | `RenewTld` | Verlängerung der 3-Jahres-Laufzeit einer PRIVATEN TLD (PoW gleich der Erstellung) |
| `0x15` | `SlashTx` | Nachweis einer Anker-Äquivokation + Ausschluss (M9) |
| `0xE2` | `TransferDomain` | Übertragung einer Domain an einen neuen Owner (M8c) |
| `0x71` | `Heartbeat` | Lebenszeichen eines Betreibers (Proof-of-Service-RFC) |
| `0xA9` | `SetTldOperator` | Delegation des Betriebs einer privaten TLD (Proof-of-Service-RFC) |
| `0x5E` | `SetDomainOperator` | Delegation des Betriebs einer Domain (Proof-of-Service-RFC) |

Jedes Versionsbyte außer `0x01` wird ausdrücklich abgelehnt
(`UnsupportedVersion(empfangener Wert)`): ein unbekanntes oder
veraltetes Format wird nie versehentlich geparst.

## SLASH_TX (0x15, M9)

On-Chain-Nachweis, dass ein Anker zwei **widersprüchliche
Checkpoints in derselben Epoche** signiert hat (Äquivokation). Jeder
kann ihn einreichen — der Nachweis ist kryptografisch und
in sich geschlossen:

```text
offender[32] | evidence_a (CheckpointData 116 B)
  | sig_a[64] | evidence_b (CheckpointData) | sig_b[64]
  | reporter_pk[32] | reporter_signature[64]
```

- `validate()` prüft die Struktur: dieselbe Epoche, verschiedene
  `signing_hash`es, zwei verschiedene Signaturen,
  Reporter-Signaturen über der kanonischen Nutzlast;
- Zustandsregeln (journliert, bei Reorg rückgängig): BEIDE
  Signaturen des Beschuldigten müssen gegen die ZWEI verschiedenen
  Signing-Hashes in derselben Epoche verifizieren; der Beschuldigte
  muss im berechtigten Pool sein (sonst
  `SlashOffenderNotInPool`); dann **Ausschluss** — der Schlüssel
  wird dauerhaft aus dem PoS-Pool entfernt (Blacklist im Zustand);
- minimale Sanktion: die Domains/TLDs des Beschuldigten werden NICHT
  eingezogen (dokumentierte M9-Entscheidung);
- derselbe Checkpoint zweimal = keine Äquivokation (abgelehnt);
  verschiedene Epochen = abgelehnt; gefälschte Signatur = abgelehnt.

## REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

| Feld | Typ | Bedingungen |
|---|---|---|
| `name` | `str` ≤ 253 | kanonischer Name der beanspruchten Domain, **im Klartext getragen** (siehe unten) |
| `domain_id` | 32 rohe Bytes | **muss** `DomainId(name)` gleichen (neu berechnet, nie geglaubt) |
| `owner` | 32 rohe Bytes | **muss** die Ableitung von `public_key` sein (siehe unten) |
| `timestamp` | varint | Reihenfolgewert (Unix, Sekunden) |
| `proof` | `bytes` ≤ 256 | Registrierungs-PoW-Nutzlast (siehe [unten](#registrierungs-proof-of-work-pow)) |
| `public_key` | 32 rohe Bytes | öffentlicher Ed25519-Schlüssel des Signierenden (RFC 8032) |
| `signature` | **genau 64 Bytes** | Ed25519-Signatur der signierten Nutzlast |

### Der Name wird von der Transaktion getragen

`RegisterDomain` transportiert den kanonischen Namen im Klartext,
gefolgt von seiner `DomainId`. Die Konsistenz `domain_id ==
DomainId(name)` wird beim Kodieren, Dekodieren und Validieren erneut
geprüft (Verteidigung in der Tiefe): ein `RegisterDomain`, dessen Id
nicht zum getragenen Namen passt, ist ungültig. Die `DomainId`
bleibt die Konsens-Identität (Chain- und DHT-Schlüssel); der
getragene Name macht die Chain selbstbeschreibend und deckt den
gesamten Claim in der Signatur ab.

## REGISTER_TLD (0x93)

Claim einer Top-Level-Domain (TLD-Register, M7):

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) open(0x00|0x01) public_key[32] signature[64]
```

| Feld | Typ | Bedingungen |
|---|---|---|
| `tld` | `str` ≤ 63 | kanonischer Name der TLD, **im Klartext getragen** (M8b: die PoW-Challenge leitet sich vom Namen ab, ebenso wie `RegisterDomain` seinen eigenen trägt) |
| `tld_id` | 32 rohe Bytes | Identität der TLD: `BLAKE3-256("SCONE-TLD-V1" || tld)` — **muss** die Ableitung von `tld` sein (neu berechnet) |
| `owner` | 32 rohe Bytes | Ableitung von `public_key` |
| `timestamp` | varint | Reihenfolgewert (Unix, Sekunden) |
| `proof` | `bytes` ≤ 256 | Registrierungs-PoW-Nutzlast (siehe [unten](#registrierungs-proof-of-work-pow)) |
| `open` | `0x00\|0x01` | **offene** Geburt: die TLD landet direkt in `TldState::Open` (niemands, Self-Service + PoW, ein einziges PoW zum Satz `open_tld_pow_difficulty` — billiger als privat: gemeinsame Ressource, kein PoS-Sitz noch Exklusivität); **private** Geburt: historisches Verhalten (Owner, PoS-Sitz, `tld_pow_difficulty`) |
| `public_key` | 32 rohe Bytes | öffentlicher Ed25519-Schlüssel des Signierenden |
| `signature` | **genau 64 Bytes** | Ed25519-Signatur der signierten Nutzlast |

Der `TldId`-Raum ist **disjunkt** vom `DomainId`-Raum (verschiedene
Ableitungspräfixe — siehe `/docs/en/general/naming.md`): die
Registrierung der TLD `uip` (ein Claim auf den Namespace `*.uip`)
kann nie die Identität einer Domain besetzen, und umgekehrt.
Zustandsregel bei der Anwendung: die TLD muss frei sein (von der
Domain-Registry getrenntes Register).

## Die M8a-Familie — Wire-Formate

Fünf Typen vervollständigen das TLD-Register und den
Domain-Lebenszyklus. Normative Wire-Formate seit M8a; **vollständige
Zustandsregeln seit M8b** (siehe
`/docs/en/technical/blockchain.md`): TLD-Owner erforderlich, PoW,
wo immer ein Claim im Self-Service läuft, Ablauf und Karenz auf
Domain-Seite. Keine Nutzlast trägt ein `network`-Feld auf dem Wire
(es lebt nur in der Signaturnutzlast).

### TRANSFER_TLD (0x47)

```text
tld_id[32] owner[32] new_owner[32] proof.v public_key[32] signature[64]
```

| Feld | Typ | Bedingungen |
|---|---|---|
| `tld_id` | 32 rohe Bytes | die übertragene TLD |
| `owner` | 32 rohe Bytes | Ableitung von `public_key` (aktueller Owner / Signer) |
| `new_owner` | 32 rohe Bytes | Identität des Empfängers (opak: jeder Wert ist syntaktisch gültig; die Bindung an die Schlüssel, die ihn ausgeben können, ist eine M8b-Zustandsregel) |
| `proof` | begrenzt (Varint ‖ Bytes) | Anti-Churn-Abtretungsbeweis, Challenge `SCONE-TLD-XFER-V1 ‖ tld_id`, flache Schwierigkeit `transfer_pow_difficulty`, Epochensalz |
| `public_key` | 32 rohe Bytes | öffentlicher Ed25519-Schlüssel des Signierenden |
| `signature` | **genau 64 Bytes** | Ed25519-Signatur der signierten Nutzlast |

Kein Zeitstempel oder Sequenz: eine TLD hat zu jedem Zeitpunkt
genau einen Owner; die zuerst angewandte Übertragung gewinnt (allein
die Chain-Reihenfolge).

### REVOKE_TLD (0x6B)

```text
tld_id[32] owner[32] public_key[32] signature[64]
```

 Freiwillige Aufgabe einer TLD (danach durch ein frisches
`RegisterTld` neu beanspruchbar). Vom aktuellen Owner signiert;
bewusst ohne Zeitstempel oder Sequenz — ein Revoke ist einmalig
gegenüber dem aktuellen Zustand.

### SET_TLD_OPEN (0xB8)

```text
tld_id[32] owner[32] open u8 open_proof(proof) public_key[32] signature[64]
```

| Feld | Typ | Bedingungen |
|---|---|---|
| `open` | 1 Byte | **strikt `0x00` oder `0x01`** (kanonische Form; jeder andere Wert → Dekodierfehler, nie Umwandlung) |
| `open_proof` | `proof` (Längen-Varint + Bytes) | **signiertes** Feld; obligatorisch und verifiziert bei `open = 0x01` (Öffnungs-PoW, `open_tld_pow_difficulty` des Netzwerks, Challenge `SCONE-TLD-OPEN-V1 ‖ tld_id`); leer bei `open = 0x00` (Schließen ist gratis) |

Eine **geschlossene** TLD = nur assign (Domains ausschließlich
durch den Eigentümer der TLD per `AssignDomain` erzeugt); eine
**offene** TLD = jeder kann das Registrierungs-PoW ausführen und
eine freie Domain beanspruchen (`RegisterDomain`). Das **Öffnen**
kostet ein zusätzliches Proof of Work (leichter als der TLD-Claim:
~3 Tage auf dem Mainnet gegenüber ~1 Woche), einmal vom Owner in
`SetTldOpen` bezahlt — der Proof reist im signierten Feld
`open_proof` und kann nach dem Signieren nicht substituiert werden.

### ASSIGN_DOMAIN (0xD4)

```text
name(str ≤ 253) domain_id[32] owner[32] assignee[32] proof.v public_key[32] signature[64]
```

Spiegel von `RegisterDomain`: der kanonische Name wird im Klartext
getragen und `domain_id` muss `DomainId(name)` gleichen (neu
berechnet beim Dekodieren, Kodieren und Validieren). Signiert vom
**Eigentümer der TLD des Namens** (bei Anwendung geprüft, M8b);
`assignee` wird erster Owner der Domain (opak, wie `new_owner`).
Trägt seinen eigenen flachen Anti-Spam-Beweis
(`assign_pow_difficulty`, Challenge `SCONE-DOMAIN-ASSIGN-V1 ‖ name`
— flach per Entscheidung: die Exklusivität des Namespace wurde bei
der TLD-Erstellung bezahlt).

### RENEW_DOMAIN (0x3C)

```text
name(str ≤ 253) domain_id[32] owner[32] valid_until.v proof(begrenzt) public_key[32] signature[64]
```

| Feld | Typ | Bedingungen |
|---|---|---|
| `name` | `str` ≤ 253 | kanonischer Name **im Klartext getragen** (Owner-Option: die Verlängerung schiebt den Tod der zugrunde liegenden offenen TLD auf, die Chain muss den Namen kennen); `domain_id` muss `DomainId(name)` gleichen (neu berechnet) |
| `valid_until` | varint | neuer Registrierungsablauf (Unix, Sekunden) |
| `proof` | begrenzt | **Verlängerungsautorisierung** (siehe unten) |

Vom aktuellen Owner der Domain signiert; ob `valid_until` das
Ablaufdatum tatsächlich verlängert (und um höchstens eine
Verlängerungsperiode) ist eine Zustandsregel (M8b).

#### Verlängerungsautorisierung — PoW oder PoS je nach TLD

Die Kosten einer Verlängerung hängen von der Natur der
zugrunde liegenden TLD ab (zur Anwendungszeit, auf dem
Blockzustand, ausgewertet):

- **Offene TLD**: die Verlängerung kostet **dasselbe PoW wie die
  Erstellung** — dieselbe Challenge (`SCONE-DOMAIN-V1 ‖ kanonischer
  Name`), dieselbe schwierigkeit pro Label-Länge und pro Netzwerk,
  dasselbe Epochen-Gültigkeitsfenster. Eine Domain wird nie gratis
  „eingesammelt“: jede Lebensspanne wird in Arbeit bezahlt.
- **Private TLD**: der Namespace ist der Einsatz des TLD-Eigentümers
  (PoS). Die Transaktion muss **vom TLD-Eigentümer (oder seinem
  delegierten Betreiber) signiert** sein — die Signatur IST die
  Autorisierung, das Feld `proof` muss leer sein (20 Null-Bytes).
  Der TLD-Eigentümer verlängert die Domains seines Namespace nach
  seinen eigenen Regeln.

### RENEW_TLD (0x8F)

```text
tld(str ≤ 63) tld_id[32] owner[32] valid_until.v proof(begrenzt) public_key[32] signature[64]
```

| Feld | Typ | Bedingungen |
|---|---|---|
| `tld` | `str` ≤ 63 | kanonischer TLD-Name; `tld_id` muss `TldId(tld)` gleichen (neu berechnet) |
| `valid_until` | varint | muss GENAU `now + 3 Jahre` sein (feste Laufzeit — bei Anwendung neu abgeleitet, jeder andere Wert abgelehnt) |
| `proof` | begrenzt | Verlängerungs-PoW (siehe unten) |

Vom aktuellen **privaten** Owner der TLD signiert.
Anwendungsregeln:

- gilt nur für eine PRIVATE TLD (eine offene TLD lebt durch ihre
  Domains — typisierte Zurückweisung `TldIsOpen`);
- gültig während der Laufzeit UND durch die **60-Tage-Karenz**, die
  dem Ablauf folgt; darüber hinaus ist die TLD tot (GC) und die
  Zurückweisung ist `TldExpiredFrozen`;
- **PoW gleich der Erstellung**: Challenge
  `SCONE-TLD-V1 ‖ Name ‖ aktuelles_Ablaufdatum(BE64)` (das
  Anti-Wiederverwendungs-Suffix bindet jedes PoW an EINE Laufzeit),
  dieselbe schwierigkeit pro Länge wie beim ersten Claim;
- die Verlängerung ist während der Karenz NICHT eingefroren — sie
  ist die einzige Verwaltungsoperation, die einem abgelaufenen
  Eigentümer bleibt.

## UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

| Feld | Typ | Bedingungen |
|---|---|---|
| `domain_id` | 32 rohe Bytes | Zieldomain |
| `owner` | 32 rohe Bytes | Ableitung von `public_key` |
| `sequence` | varint | **> 0**, muss bei Anwendung `current + 1` gleichen |
| `record_hash` | 32 rohe Bytes | Commitment auf den DHT-Eintragssatz |
| `public_key` | 32 rohe Bytes | öffentlicher Ed25519-Schlüssel des Signierenden |
| `signature` | **genau 64 Bytes** | Ed25519-Signatur der signierten Nutzlast |

Hinweis: `UpdateDomain` trägt **keinen** `timestamp` — die
Reihenfolge läuft über `sequence`. Es trägt auch nicht den Namen:
die Domain ist bereits registriert, die Id genügt.

### Strenge Grenzen

- `public_key`: **genau 32 rohe Bytes**, kein Längenpräfix. Ein pk,
  der nicht als Kurvenpunkt dekodiert → Fehler `InvalidTransactionKey`
  beim Dekodieren.
- `signature`: **genau 64 rohe Bytes**, kein Längenpräfix, kein
  Spielraum. Jede andere Größe ist ein harter Dekodierfehler
  (Abschneidung abgelehnt).
- `proof`: Längen-Varint + Bytes, begrenzt auf 256
  (`MAX_PROOF_LEN`), vor der Allokation geprüft.
- `name`: Längen-Varint + UTF-8-Bytes, begrenzt auf 253
  (`MAX_NAME_LEN`), durch die Namensregeln von `scone-core`
  revalidiert.

## POW-TO-OWNER — vier signaturlose Formen (0x2A, 0x57, 0x84, 0xC3)

Vier ADDITIVE Varianten neben den signierten Formen (nie ein
Ersatz — D1): `REGISTER_TLD_POW` (0x2A), `RENEW_TLD_POW`
(0x57), `REGISTER_DOMAIN_POW` (0x84), `RENEW_DOMAIN_POW` (0xC3).
Keine trägt einen Signer-Schlüssel NOCH eine Signatur: **der
geminte Proof of Work ist die gesamte Autorisierung**.

### Die Challenge bindet den Begünstigten und das Netzwerk (D2)

```
challenge = kind_tag ‖ name ‖ pubkey_dest[32] ‖ len(network_id) ‖ network_id
```

- `kind_tag`: das Familien-Tag (SCONE-TLD-V1 für TLDs,
  SCONE-DOMAIN-V1 für Domains) — kein Cross-Kind-Replay;
- `pubkey_dest`: der Schlüssel des BEGÜNSTIGTEN — ein
  übergebener Beweis ist nur von diesem Schlüssel konsumierbar
  (kein Beweis-Diebstahl, sicheres delegiertes Mining);
- `network_id`: ein Devnet-Beweis wird nie auf Testnet oder
  Mainnet wiederholt.

Die RENEWAL-Formen hängen das Anti-Wiederverwendungs-Suffix der
signierten Familie an: `‖ current_valid_until` (BE, 8 Bytes) —
ein geminter Beweis bezahlt genau eine Laufzeit.

### Körperstruktur

Exakte Spiegel der signierten Körper, `pubkey_dest` an Stelle des
Signer-Schlüssels, KEINE abschließende Signatur:

- `REGISTER_TLD_POW`: name ‖ tld_id ‖ owner ‖ timestamp.v ‖ proof ‖
  open(0x00|0x01) ‖ pubkey_dest — `owner` = hash(pubkey_dest),
  neu berechnet, nie geglaubt (D4); open = OPEN-Geburt (geringerer
  Satz); private = strenge 3-Jahres-Laufzeit, der Begünstigte
  wird ein PoS-Sitz;
- `REGISTER_DOMAIN_POW`: name ‖ domain_id ‖ owner ‖ timestamp.v ‖
  proof ‖ pubkey_dest — nur offene TLD (private → TldClosed: der
  private Namespace behält den Owner-Signatur-Pfad);
- `RENEW_TLD_POW` / `RENEW_DOMAIN_POW`: name ‖ id ‖ owner ‖
  valid_until.v ‖ proof ‖ pubkey_dest.

### Anwendungs-Wächter (D3)

- Der On-Chain-Owner MUSS AUCH der Begünstigte der Challenge
  sein: eine Verlängerung wechselt NIE die Hände — nur ein
  Register schafft neues Eigentum;
- Schwierigkeiten, Epochenfenster, Gnadenfrist und Laufzeiten sind
  die der SIGNIERTEN Formen selbst (D5: die Eintrittskosten
  ändern sich nicht);
- der Schlüssel des Begünstigten wird ermittelt (PoS-Pool-Auflösung)
  genau wie der eines Signers.

### txid und Koexistenz

Die txid bleibt der kanonische Hash des Inhalts; die
0x2A/0x57/0x84/0xC3-Diskriminanten sind die Fork-Markierung — ein
alter Knoten lehnt sie als UnknownDiscriminant ab, ein neuer
Knoten dekodiert die v1-Formen unverändert. `signing_payload` ist
N/V für alle vier (typisierter `PowNotSignable`-Fehler): sie
werden nie signiert.

CLI: `scone domain register <name> --to <pubkey-hex>`,
`scone domain mine <name> --mine-for <pubkey>`,
`scone domain renew <name> --to`, `scone tld register/renew --to`.

## Owner-Ableitung

Harte Regel: **der Owner wird immer neu berechnet, nie geglaubt**.

```text
PublicKeyRef = BLAKE3-256("SCONE-PUBKEY-V1" || public_key)
OwnerId      = BLAKE3-256("SCONE-OWNER-V1" || PublicKeyRef)
```

Eine Transaktion, deren `owner` nicht zur Ableitung ihres eigenen
`public_key` passt, ist ungültig (`InvalidOwner` auf Core-Seite,
`OwnerKeyMismatch` auf Chain-Seite), zurückgewiesen beim Kodieren,
Dekodieren **und** bei der Chain-Validierung (Verteidigung in der
Tiefe).

## Signierte Nutzlast

Die signierte Nutzlast ist die kanonische Kodierung der Transaktion
**ohne das Signaturfeld**, mit einem
Domänen-Trenn-Tag als Präfix — **mit der `network_id` der
Ziel-Chain am Kopf injiziert**:

```text
signing_payload(tx, network) = "SCONE-TX-SIG-V1"
                              || canonical_encode(tx_without_signature, network)
```

wobei `canonical_encode(tx_without_signature, network)` der
Wire-Struktur oben folgt, Diskriminant und Versionsbyte
eingeschlossen, haltend vor `signature`, **mit `network` als erstem
Feld** — dies ist der EINZIGE Ort, an dem das Netzwerk erscheint.
Die Chain berechnet die Nutzlast mit IHREM eigenen Netzwerk neu:
eine für ein anderes Netzwerk signierte Transaktion scheitert an
der Signaturverifikation (`InvalidSignature`); das ist der
Anti-Cross-Chain-Replay-Mechanismus.

```text
REGISTER_DOMAIN_no_sig = disc(0x21) version(0x01) network name domain_id[32] owner[32]
                            timestamp.v proof public_key[32]
REGISTER_TLD_no_sig    = disc(0x93) version(0x01) network tld tld_id[32] owner[32]
                            timestamp.v proof public_key[32]
UPDATE_DOMAIN_no_sig   = disc(0x52) version(0x01) network domain_id[32] owner[32]
                            sequence.v record_hash[32] public_key[32]
TRANSFER_TLD_no_sig    = disc(0x47) version(0x01) network tld_id[32] owner[32]
                            new_owner[32] proof public_key[32]
REVOKE_TLD_no_sig      = disc(0x6B) version(0x01) network tld_id[32] owner[32]
                            public_key[32]
SET_TLD_OPEN_no_sig    = disc(0xB8) version(0x01) network tld_id[32] owner[32]
                            open(0x00|0x01) open_proof(proof) public_key[32]
ASSIGN_DOMAIN_no_sig   = disc(0xD4) version(0x01) network name domain_id[32] owner[32]
                            assignee[32] proof public_key[32]
RENEW_DOMAIN_no_sig    = disc(0x3C) version(0x01) network name domain_id[32] owner[32]
                            valid_until.v proof public_key[32]
TRANSFER_DOMAIN_no_sig = disc(0xE2) version(0x01) network domain_id[32] owner[32]
                            new_owner[32] proof public_key[32]
```

Eigenschaften:

- deterministisch und rein: dieselbe Transaktion ⇒ dieselbe
  Nutzlast überall;
- das ASCII-Präfix `SCONE-TX-SIG-V1` (15 Bytes,
  hex `53434f4e452d54582d5349472d5631`) verhindert jede
  Wiederverwendung der Signatur in einer anderen Hash-Domäne des
  Protokolls;
- alle Felder außer `signature` sind abgedeckt — **einschließlich
  des von einem `RegisterDomain` getragenen Namens**: die Änderung
  eines einzigen von ihnen macht die Signatur ungültig;
- die injizierte `network_id` ist abgedeckt: dieselbe unsignierte Tx
  für zwei Netzwerke signiert erzeugt zwei verschiedene Nutzlasten
  (und damit zwei verschiedene Signaturen).

## Registrierungs-Proof of Work (PoW)

Das Feld `proof` von `RegisterDomain` und `RegisterTld` trägt das
Registrierungs-Proof of Work (Modul `scone-core::pow`).

### Digest (Epochen-Salz, Anti-Pre-Mining)

```text
epoch  = block_height / POW_WINDOW        (POW_WINDOW = 144 Blöcke)
salt   = BLAKE3-256("SCONE-POW-V1-ARGON2" || network || challenge || epoch_le64)
digest = Argon2id(m, t, p = 1; password = nonce_le64, salt = salt[0..16])
```

`network` ist die kanonische `network_id` (M8b): ein für ein
Netzwerk gemintes Proof ist auf einem anderen kein Proof.

`SCONE-POW-V1-ARGON2` hebt außerdem die Argon2id-Parameter pro
Netzwerk fest (`pow_params`: Mainnet m = 64 MiB, t = 2, p = 1;
Testnet/Devnet m = 8 KiB, t = 1, p = 1): der Argon2id-Kern
(RFC 9106) macht die Kosten pro Hash speicherdominiert
(GPU/ASIC-Resistenz), und ein unter einem Parametersatz gemintes
Proof verifiziert nie unter einem anderen.

`challenge` ist der Ableitungseingang des beanspruchten Objekts —
`"SCONE-TLD-V1" || tld` für eine TLD, `"SCONE-DOMAIN-V1" || name`
für eine Domain: exakt die Bytes, die für `TldId` / `DomainId`
gehasht werden, sodass ein Proof weder über Namespaces noch über
Objektarten wiederspielbar ist. `epoch_le64` und `nonce_le64` sind
die Little-Endian-8-Byte-Kodierungen der Epoche und der Nonce.

Das Proof ist gültig, wenn `digest` mindestens `difficulty`
führende Nullbits hat.

#### Epochen-Salz — Anti-Pre-Mining

Ein ungesalzener Digest bindet weder Zeit noch Höhe: jeder könnte
heute Proofs für Tausende Namen vorminen und sie beim
Mainnet-Start in einem Burst registrieren (Squatting) — das Proof
war in sich geschlossen und gültig, egal welcher Block zuerst kam.
Der Digest wird mit der **Epoche des Blocks gesalzen, der die
Transaktion ANWENDET**:

- `epoch = block_height / POW_WINDOW`, `POW_WINDOW = 144` Blöcke
  (~2,6 h bei 65 s/Block): ein Kulanzfenster für den ehrlichen
  Nutzer (minen, dann innerhalb des Fensters einreichen),
  obrigenses Neu-Minen jenseits davon;
- das Proof DEKLARIERT seine Epoche auf dem Wire; bei der Anwendung
  muss die deklarierte Epoche ins **Gültigkeitsfenster** des
  anwendenden Blocks fallen — NUR rückwärtige Toleranz
  (verspätete Einreichung wird aufgenommen, Minen „für die Zukunft“
  ist unmöglich), Fenstergröße nach
  `epoch_validity_window(difficulty)` = `1 << max(0, difficulty −
  16)` Epochen (erweitertes Fenster: hohe Mainnet-Schwierigkeiten
  werden in Tagen-bis-Wochen gemint, länger als eine Epoche; das
  Horten bringt nichts, weil das Fenster ≈ die Mining-Kosten ist —
  jedes Proof eines Bursts kostet seine vollen `2^difficulty` Hashes
  INNERHALB des Fensters). Bei schwierigkeit ≤ 16 Bit (Testnet,
  lange Namen) bleibt die Regel der strikten Gleichheit unverändert;
- Mempool-Zulassung (best effort): die zukünftige Blockhöhe ist
  unbekannt — das Relay verifiziert gegen die Epoche von
  `aktuelle Höhe + 1`; die strikte Verifikation zur Anwendungszeit
  bleibt die Autorität;
- deterministisches Replay: das Proof TRÄGT seine Epoche und das
  Fenster ist eine reine Funktion von (Schwierigkeit,
  Block-Epoche) — ein historischer Block verifiziert auf jedem
  Knoten identisch neu.

Breaking Change vor dem Mainnet: das alte kompakte Format (12 Bytes)
wird ohne Rückwärtskompatibilität abgelehnt.

### Wire-Nutzlast (20 feste Bytes)

```text
nonce_le64 || epoch_le64 || difficulty_le32
```

Die Schwierigkeit und die Epoche reisen mit dem Proof, sind aber
**vom Produzenten deklariert und daher nie geglaubt**: der Prüfer
vergleicht die Schwierigkeit mit der **Protokollkonstante** für die
Registrierungsart (eine Nutzlast, deren deklarierte Schwierigkeit
von der Konstante abweicht — niedriger *oder* höher — wird
abgelehnt, `InvalidProof`), und die Epoche mit der des anwendenden
Blocks (`ProofEpochMismatch` bei Abweichung). Die Nonce wird dann
auf der konstanten Schwierigkeit neu verifiziert, der Digest von
Grund auf neu berechnet.

### Schwierigkeiten (**längenbasierte** Kurve, M8b + gemessene Kalibrierung)

Die Schwierigkeit ist KEIN flacher Parameter: sie ist eine
deterministische Funktion der **Länge des erzeugten Labels**
(`scone-core::pow_length` — die TLD für `RegisterTld`, das
linksstehende Label für `RegisterDomain`), pro Netzwerk:

```text
difficulty(len) = apex − round(0.415 × (len − 1)), an der Untergrenze geklemmt
```

| Netzwerk | Private TLD (Apex/Floor) | Offen geborene TLD / Domain (Apex/Floor) |
|---|---|---|
| testnet | 8 Bits / 2 Bits | 6 Bits / 2 Bits |
| mainnet | 42 Bits / 30 Bits | 40 Bits / 30 Bits |
| devnet | 2 Bits / 1 Bit | 1 Bit / 1 Bit |

Eine **offen geborene** TLD (`RegisterTld` mit `open = 0x01`) zahlt
die „Öffnungs“-Kurve (Apex − 2, wie `domain_difficulty`); das
**Öffnen einer bestehenden privaten TLD** (`SetTldOpen`) bleibt auf
flacher Schwierigkeit (`open_tld_pow_difficulty`: 6 Bits Testnet /
22 Bits Mainnet / 2 Bits Devnet — die Tx trägt nur die `tld_id`, der
Name hat seine Länge bei der Erstellung bezahlt).
Mainnet-Benchmarks (private TLD, ~6 M Hashes/s): 1 Zeichen = 42
Bits (≥ 7 Tage), 5 Zeichen = 40 Bits (~2 Tage), 19 Zeichen = 35
Bits (~1,6 h), 29+ Zeichen = 30-Bit-Floor (~3 Min). Die Kurven
leben in `scone-core::pow_length` (Konstanten `MAINNET_APEX = 42`,
`MAINNET_FLOOR = 30`, `TESTNET_APEX = 8`, `TESTNET_FLOOR = 2`,
`DEVNET_APEX = 2`, `DEVNET_FLOOR = 1`); die flachen Parameter
(`tld_pow_difficulty` usw.) bleiben in
`scone-core::network::NetworkParams`. Ein Wert ändern = ein
Konsenswechsel **für dieses Netzwerk**. Invariante: das Devnet ist
ein vollständiges drittes `NetworkParams`, und das Testnet bleibt
strikt identisch zu seinen Prä-Devnet-Konstanten
(Argon2id-Parameter, Kurven, Fenster) — das Devnet teilt KEINE
Konstante mit den anderen Netzwerken über den Code hinaus.

Gemessene Kalibrierung der flachen Gipfel (2026-10, `cargo run
--release -p scone-core --example pow_bench` — Single-Thread-Durchsatz
~7,7 M H/s auf einer Referenz-x86-64):

| Art (Mainnet) | Schwierigkeit | durchschnittliche reale Zeit |
|---|---|---|
| `RegisterTld` | 24 Bits | ~7 s |
| `RegisterDomain` | 20 Bits | ~54 ms |

Bei ~65 s/Block: ein TLD-Claim kostet ~0,1 Block Single-Core-CPU,
eine Domain ~1 ms Block — symbolisch für einen Nutzer, teuer zu
industrialisieren bei Skalierung (Anti-Squat). Neu zu kalibrieren,
falls Hardware-Ziele sinken (Mobile-ARM: ~4–8× langsamer).

Zustandsverzweigungen (M8b, ausgeliefert): `RegisterTld` verlangt
IMMER sein PoW; `RegisterDomain` verlangt es auf einer **offenen**
TLD (der geschlossene Pfad ist nur assign — `AssignDomain`, vom
Eigentümer der TLD signiert, mit seinem eigenen flachen
Anti-Spam-Beweis).

**Abtretungs- und Assign-Beweise (Konsens-Anti-Spam)**:
`TransferTld`, `TransferDomain` und `AssignDomain` tragen jeweils
einen **flachen** Proof of Work — `transfer_pow_difficulty` für die
zwei Abtretungen (Challenges `SCONE-TLD-XFER-V1 ‖ tld_id` /
`SCONE-DOMAIN-XFER-V1 ‖ domain_id`), `assign_pow_difficulty` für
den Assign (Challenge `SCONE-DOMAIN-ASSIGN-V1 ‖ name`). Dasselbe
Epochensalz und Anti-Premining-Fenster wie die Registrierungen;
verifiziert bei der Mempool-Zulassung (geschätzte Epoche,
Best-Effort), dann bei der Anwendung (streng). Ziel: Transfer-Churn
und Massen-Erstellung unter einer privaten TLD kosten Arbeit auf
Konsensebene, nicht Politik. Keine Längen-Besteuerung: für Assigns
wurde die Exklusivität des Namespace bereits bei der Erstellung der
privaten TLD bezahlt.

**Passive Abtretung**: diese Erwerbswege (`TransferTld`,
`TransferDomain`, `AssignDomain`) werden allein vom Übertragenden
signiert — der Empfänger wird nie konsultiert. Eine unterstellte
Wahl (Geschenke, Dead Drops, Vermächtnisse), ohne auferlegbare
PoS-Schuld: siehe `security-model.md` (passiver Erwerb und
Berechtigung).

## Validierungsregeln

Eine Transaktion ist genau dann gültig, wenn:

1. **striktes Dekodieren**: Struktur und Grenzen oben eingehalten
   (bekannter Diskriminant, Version `0x01`, pk 32 B, Signatur 64 B,
   Core-Invarianten beim Dekodieren erneut geprüft);
2. **Owner/pk-Bindung**: `owner == OwnerId::from(public_key)`
   (neu berechnet);
3. **Name/Id-Konsistenz** (RegisterDomain): `domain_id ==
   DomainId(name)` (neu berechnet);
4. **Signatur**: Ed25519 `verify_strict` von `signature` gegen die
   **von Grund auf neu berechnete** signierte Nutzlast mit dem
   Netzwerk der Chain (nie von einem Peer oder aus der Transaktion
   wie geliefert entnommen) — eine für ein anderes Netzwerk
   signierte Tx scheitert hier (`InvalidSignature`,
   Anti-Cross-Chain-Replay);
5. **Zustandsregeln** (bei Anwendung in einem Block — M8b, siehe
   `/docs/en/technical/blockchain.md`):
   RegisterTld → TLD frei + nicht ICANN + PoW (längenbasierte
   Schwierigkeit); RegisterDomain → TLD registriert **und offen** +
   freie Domain (Karenz eingeschlossen) + PoW (längenbasiert);
   UpdateDomain → existierende Domain, aktueller `owner`,
   `sequence == current + 1`;
   TransferTld / RevokeTld / SetTldOpen / AssignDomain →
   existierende **und private** TLD, Signer = Eigentümer der TLD;
   TransferDomain → existierende Domain, Owner;
   RenewDomain → existierende Domain, strikte Verlängerung ≤ 3
   Jahre, PoW-Autorisierung (offene TLD: gleiches PoW wie die
   Erstellung, **an das aktuelle Ablaufdatum gebunden** — eine
   Challenge, verschieden von der Registrierung und jeder früheren
   Verlängerung) oder PoS (private TLD: Signatur des TLD-Eigentümers
   — oder seines effektiven Betreibers —, leeres `proof`).

Die Verifikation verwendet `verify_strict` (Zurückweisung malleabler
Signaturen und schwacher/nicht-kanonischer Schlüssel), wie für
unvertrauenswürdige Daten erforderlich.

## Reihenfolge der Operationen in `push_block`

Die kryptografische Validierung (2 bis 4) findet für jede
Transaktion **vor** der Zustandsanwendung statt, auf einem atomaren
Arbeitszustand pro Block; jeder Fehler lässt die Chain strikt
unverändert.

## Blockzusammenstellung

`BlockBuilder` (`scone-blockchain`) setzt einen Block aus einer
Transaktions-Warteschlange zusammen: `tx_root` über der geordneten
Liste neu berechnet, Header an den Parent-Hash gekettet
(`prev_hash`), `version` = `PROTOCOL_VERSION`. Die
Konsensnutzlast (signierte Produzentennutzlast) und der Zeitstempel
werden vom Aufrufer geliefert; der bestehende Trait `Consensus` wird
symmetrisch zur Validierung verwendet.

## Hex-Beispiele (echte, reproduzierbare Vektoren — M8b)

### RegisterDomain `example.uip`, Zeitstempel 42, Testnet

Signierte Nutzlast (angehefteter Vektor; historisch erzeugt von der
entfernten Offline-Toolchain — die Werte bleiben gültige Vektoren).
Das `network` erscheint hier, weil dies die SIGNIERNUTZLAST ist (ihr
einziger Ort); die signierte Wire-Transaktion selbst enthält keines:

```text
53434f4e452d54582d5349472d5631       SCONE-TX-SIG-V1 prefix
21                                    disc REGISTER_DOMAIN (0x21)
01                                    format version
0d 73636f6e652d746573746e6574         network (str, "scone-testnet")
0b 6578616d706c652e756970             name (str, length 11)
e6a2cbe264f81ed7da4d901c4deec6a3579224b8080fad01cb7461b8c264f181  domain_id
7b2446dadddf640244ac8677a401f148b2b23d1c836d5ad3dd340f8ef2edbed8   owner (fixed build key)
2a                                    timestamp (varint, 42)
14 0200000000000000 0000000000000000 04000000
                                      proof (20 B: nonce 2, epoch 0, difficulty 4 — testnet)
2a002152f8d19b791d24453242e15f2eab6cb7cffa7b6a5ed30097960e069881db12  pk + start of sig
```

### RegisterTld `uip`, Zeitstempel 42

Signierte Nutzlast (angehefteter Vektor) — die `TldId` ist der in
`/docs/en/general/naming.md` angeheftete Ableitungsvektor:

```text
53434f4e452d54582d5349472d5631       SCONE-TX-SIG-V1 prefix
93 01                                 disc REGISTER_TLD + version
0d 73636f6e652d746573746e6574         network (str, "scone-testnet")
03 756970                             tld (str, "uip")
ad5a86d68643d5c22d6a959bb1a315530c77dfda241f1ac1a8107450f3fab25e  tld_id ("uip")
7b2446dadddf640244ac8677a401f148b2b23d1c836d5ad3dd340f8ef2edbed8   owner
2a                                    timestamp (varint, 42)
14 8e00000000000000 0000000000000000 07000000
                                      proof (20 B: nonce 0x8e, epoch 0, difficulty 7 — testnet, 3-char label)
00                                    open (0x00 = private birth)
2a002152f8d19b791d24453242e15f2eab6cb7cffa7b6a5ed30097960e069881db12  pk
```

(Die vollständigen Werte waren über die entfernte Offline-Toolchain
reproduzierbar; die Vektoren unten bleiben angeheftet und gültig.
Das PoW wird automatisch auf Netzwerk-Schwierigkeit gemint — Nonces
variieren von Lauf zu Lauf; nur network/name/id/owner/pk sind
stabil. Seit der Wire-Änderung 2026-09 enthält das Hex der
signierten Transaktion NICHT MEHR die Bytes
`0d 73636f6e652d746573746e6574`: das Netzwerk reist nur in der zu
signierenden Nutzlast.)

## Wo das Hex jetzt lebt

Die Offline-Befehle `tx build|sign|verify` wurden entfernt
(cli-simplify-v1). Das Signiert-Hex-Format selbst ist unverändert
und bleibt das Wire-Format des **Web/API-Flows**: die
Registrar-Web-App baut, signiert (Keystore im Browser) und reicht
Transaktionen als kanonisches Hex über die API ein. Auf
CLI-Seite sind die Nutzerpfade die Ein-Befehl-Formen:

```bash
scone domain register example.uip        # mint, signiert, reicht ein
scone domain renew example.uip                      # Verlängerungs-Proof, signiert, reicht ein
scone domains                            # was Sie besitzen
```

In allen Codepfaden werden Owner und Schlüssel aus der
Signieridentität **neu berechnet**, nie aus einer Eingabe
übernommen.

## Versionsgeschichte

| Version | Inhalt | Status |
|---|---|---|
| 1 | eingebettetes `public_key` + `signature`, Nutzlast `SCONE-TX-SIG-V1`, Versionsbyte `0x01`, `RegisterDomain` trägt den Namen, `RegisterTld` (0x93), M8a-Familie: `TransferTld` (0x47), `RevokeTld` (0x6B), `SetTldOpen` (0xB8), `AssignDomain` (0xD4), `RenewDomain` (0x3C) — normative Wire-Formate; M8b: `network`-Feld am Kopf jeder Nutzlast signiert, `RegisterTld` trägt den Namen, vollständige Zustandsregeln, PoW-Schwierigkeiten pro Netzwerk | abgelöst |
| 1 (2026-09, Wire ohne network) | **Entfernung des `network`-Feldes vom Wire aller Transaktionen**: die network_id lebt nur in der Signaturnutzlast (`signing_payload(tx, network)` injiziert sie am Kopf); die Netzwerktrennung beruht auf der Signatur (`InvalidSignature` auf einer fremden Nutzlast, der Fehler `WrongNetwork` verschwindet); `TX_FORMAT_VERSION` bleibt `0x01` (relaunchbares Testnet, keine Koexistenz); **alle TxIds ändern sich** (die TxId ist der Hash der signierten Wire-Kodierung, die 14–15 Bytes verliert); PoW-Digest, Genesis und Datenverzeichnisse pro Netzwerk unverändert | abgelöst |
| 1 (2026-10, PoW-Epochen-Salz) | **Anti-Pre-Mining-PoW-Digest**: Salz `BLAKE3("SCONE-POW-V1-ARGON2" ‖ network ‖ challenge ‖ epoch_le)` + Argon2id-Kern über `nonce_le`, mit `epoch = block_height / 144` des Blocks, der die Tx ANWENDET; Wire-Nutzlast des Proofs 12 → **20 Bytes** (`nonce_le ‖ epoch_le ‖ difficulty_le`), die deklarierte Epoche wird gegen das Epochen-Gültigkeitsfenster des anwendenden Blocks verifiziert (typisierte Zurückweisung `ProofEpochMismatch`), best-effort `(height+1)/144` bei der Mempool-Zulassung, `--epoch`-CLI-Flagge auf Offline-Pfaden; Breaking Change vor dem Mainnet: die alte 12-Byte-Nutzlast wird abgelehnt, **alle TxIds ändern sich** (das Feld `proof` wächst um 8 Bytes) | aktuell |

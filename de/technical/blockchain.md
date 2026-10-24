# Blockchain

Referenzdokument für die Blockchain-Schicht: Identitäten, Hashing,
Genesis, kanonischer Zustand, Transaktionsreihenfolge und
Blockvalidierung. Die Referenzimplementierung ist das Crate
[`scone-blockchain`](../../../crates/scone-blockchain), aufbauend auf
`scone-protocol` (kanonische Kodierung), `scone-core`
(Geschäftsregeln) und `scone-crypto` (BLAKE3).

Aktueller Umfang: Blockchain-Logik mit echtem Konsens (signierter
Produzent, Checkpoints, Finalität), **In-Memory-Zustand** über
einem KV-Backend (die konkrete Speicherung lebt in
`scone-storage`) — siehe [An Konsens zurückgestellt](#an-konsens-zurückgestellt).

## Identitäten und Hash-Domänen-Trennung

Vier Identitäten, vier Rollen, vier verschiedene BLAKE3-Präfixe —
ein Präfix wird nie über zwei Verwendungen hinweg wiederverwendet:

| Typ | Rolle | Präfix | Formel |
|---|---|---|---|
| `DomainId` | Identität des Domainnamens | `SCONE-DOMAIN-V1` | siehe `/docs/en/general/naming.md` |
| `RecordHash` | Identität des DNS-Inhalts | `SCONE-RECORD-V1` | siehe `/docs/en/technical/protocol.md` |
| `TxId` | Identität der Transaktion | `SCONE-TX-V1` | unten |
| `BlockHash` | Identität des Blocks | `SCONE-BLOCK-V1` | unten |

Der Merkle-Baum verwendet sein eigenes Präfix `SCONE-MERKLE-V1`.
Diese Präfixe kommen zu denen von `scone-core` (`SCONE-PUBKEY-V1`,
`SCONE-OWNER-V1`) hinzu.

Vollständige Abhängigkeitskette:

```text
DomainId
    ↓
RegisterDomain / UpdateDomain
    ↓
Transaction
    ↓
TxId
    ↓
MerkleRoot
    ↓
BlockHeader
    ↓
BlockHash
    ↓
prev_hash des nächsten Blocks
    ↓
kanonische Chain
```

## TxId

```text
TxId = BLAKE3-256("SCONE-TX-V1" || canonical_encode(Transaction))
```

- deterministisch: dieselbe Transaktion (dieselbe kanonische
  Kodierung) ⇒ dieselbe `TxId`, auf allen Knoten, für immer;
- Identität **durch Inhalt**: jedes geänderte Feld ändert die `TxId`
  (Domain, Owner, Zeitstempel, Proof, Sequenz, record_hash);
- kein lokaler Zeitstempel oder nicht-kanonischer Wert geht in die
  Berechnung ein;
- keine Abhängigkeit von Speicherung oder Netzwerk;
- die Identität garantiert keine Einzigartigkeit der Anwendung: das
  Abspielen einer identischen Transaktion scheitert an
  Zustandsregeln (doppeltes `RegisterDomain`, Sequenz-Replay),
  nicht an der `TxId`.

API: `scone_blockchain::transaction_id(&Transaction) -> Result<TxId>`.

## Merkle-Baum (tx_root)

Commitment über die **geordnete Liste** der Transaktionen eines
Blocks:

```text
0 Transaktionen  root = BLAKE3-256("SCONE-MERKLE-V1" || "EMPTY")
Blatt            BLAKE3-256("SCONE-MERKLE-V1" || 0x00 || TxId)
innerer Knoten   BLAKE3-256("SCONE-MERKLE-V1" || 0x01 || left || right)
ungerade Ebene   der letzte Hash wird dupliziert, um das Paar zu vervollständigen
```

Regeln:

- **niemals sortiert**: die Reihenfolge der Transaktionen in einem
  Block ist bedeutsam; zwei Blöcke mit denselben Transaktionen in
  anderer Reihenfolge haben verschiedene Wurzeln;
- die Tags `0x00` / `0x01` verhindern jede Verwechslung zwischen
  Blatt und innerem Knoten (Zweit-Preimage);
- das Präfix `SCONE-MERKLE-V1` isoliert den Baum von den anderen
  Hash-Domänen (eine `TxId` kann nicht als Knoten interpretiert
  werden);
- Eigenimplementierung (~30 Zeilen), deterministisch und getestet;
  keine externe Merkle-Bibliothek.

API: `merkle_root(&[TxId]) -> MerkleRoot` und
`tx_root(&[Transaction]) -> Result<MerkleRoot>`.

## BlockHash

```text
BlockHash = BLAKE3-256("SCONE-BLOCK-V1" || canonical_encode(BlockHeader))
```

Der Hash deckt **nur** den kanonischen Header ab: die Transaktionen
sind bereits durch `tx_root` committet, sie werden nicht ein zweites
Mal gehasht:

```text
Transaktionen -> TxIds -> MerkleRoot -> BlockHeader -> BlockHash
```

Die Änderung einer Transaktion, ihrer Reihenfolge, der Höhe, des
Zeitstempels, des `prev_hash` oder der Konsensnutzlast ändert den
`BlockHash`.

API: `block_hash(&BlockHeader) -> Result<BlockHash>`.

## Netzwerke und Genesis (M8b)

Scone definiert drei disjunkte Netzwerke, identifiziert durch eine
kanonische `network_id` (1..=16 ASCII-Bytes `[a-z0-9-]`):

| Netzwerk | `network_id` | private TLD Apex | offene TLD Apex | Domain-Apex | TLD-Floor | Domain-Floor |
|---|---|---|---|---|---|---|
| testnet | `scone-testnet` | 8 Bits | 7 Bits | 6 Bits | 2 Bits | 1 Bit |
| mainnet | `scone-mainnet` | 28 Bits | 27 Bits | 26 Bits | 25 Bits | 22 Bits |
| devnet | `scone-devnet` | 2 Bits | 1 Bit | 1 Bit | 1 Bit | 1 Bit |

Die Parameter leben in `scone-core::network::NetworkParams`
(benannte Instanzen `TESTNET` / `MAINNET` / `DEVNET`, Nachschlag
über `NetworkParams::by_name`). Eine Schwierigkeit ändern = ein
Konsenswechsel **für dieses Netzwerk**.

**Strukturelle Invariante**: das Devnet ist ein vollständiges
drittes `NetworkParams` — nie eine Teilmenge der Testnet-Flags. Das
Testnet bleibt **strikt identisch** zu dem, was es vor dem Hinzufügen
des Devnets war (gleiche Argon2id-Konstanten, gleiche Kurven,
gleiche Epochenfenster); die Testsuite heftet diese Invariante fest.

**Längenvariable Schwierigkeit** (`scone-core::pow_length`): die
Schwierigkeit einer Erzeugung hängt von der Länge des **erzeugten
Labels** ab (die TLD für `RegisterTld` — privat oder neu geöffnet —,
das linksstehende Label für `RegisterDomain`). Die Formel ergibt:

```text
TLD     : difficulty(len) = apex − round(0.287 × (len − 1)), an der Untergrenze geklemmt
domain  : difficulty(len) = apex − round(0.480 × (len − 1)), an der Untergrenze geklemmt
```

Zwei verschiedene Steigungen: `log₂(1.22) ≈ 0.287` Bit/Zeichen für
TLDs (jedes zusätzliche Zeichen teilt die Kosten durch ~1.22), und
`log₂(1.40) ≈ 0.480` Bit/Zeichen für Domains — Domains zerfallen
schneller: Ihr Wert steckt in der TLD, die sie trägt. Eine offen
geborene TLD liegt beim privaten Apex − 1; eine Domain beim privaten
Apex − 2 (Mainnet: private TLD 25, offene TLD 24, Domain 23).
Mainnet-Benchmarks (Memory-Hard-Argon2id, ~11.4 H/s Single-Thread):
private TLD 1 Zeichen = 28 Bits ≈ 30 Tage, 9+ Zeichen = 25-Bit-Floor
≈ 3,8 Tage; offene TLD 1 Zeichen = 27 Bits ≈ 15 Tage; Domain
1 Zeichen = 26 Bits ≈ 3,8 Tage, 9+ Zeichen = 22-Bit-Floor ≈ 11,5 h.
**8-Kern-Referenzmaschine** (Rekalibrierung 2026-10 — die CLI mint
Multi-Core), ~11.4 H/s/Kern. Das **Öffnen** einer bestehenden
privaten TLD (`SetTldOpen`) bleibt auf flacher Schwierigkeit
(`open_tld_pow_difficulty`: 6 Bits Testnet / 22 Bits Mainnet): die
Tx trägt nur die `tld_id`, der Name hat seine Länge bei der
Erstellung bezahlt. **Zwei weitere flache Beweise** vervollständigen
die Familie (Konsens-Anti-Spam, dasselbe Epochensalz und Fenster wie
die Registrierungen): `transfer_pow_difficulty` für Abtretungen
(`TransferTld`/`TransferDomain`, Challenges `SCONE-TLD-XFER-V1 ‖
tld_id` / `SCONE-DOMAIN-XFER-V1 ‖ domain_id`) und
`assign_pow_difficulty` für die Erstellung unter einer privaten TLD
(`AssignDomain`, Challenge `SCONE-DOMAIN-ASSIGN-V1 ‖ name` — flach
per Entscheidung: die Exklusivität des Namespace wurde bei der
TLD-Erstellung bezahlt, keine Doppelbesteuerung nach Länge).
Orientierungswerte: 16 Bits Mainnet (bench-pow-Kalibrierung
2026-10-23: ≈ 1,7 h pro Abtretung/Assign auf EINEM Kern bei
gemessenen 10.6 H/s, Referenz 11.4 H/s; ≈ 25 min bei 4 Kernen —
teuer zu industrialisieren, gewöhnliche Arbeit für einen ehrlichen
Owner), 4 Bits Testnet, 1 Bit Devnet. Reine Funktion
(Determinismus).

Die Genesis ist eine Protokollkonstante **pro Netzwerk**,
reproduzierbar aus den Konstanten allein (kein Zufall, keine lokale
Uhr, kein Netzwerk):

| Feld | Wert |
|---|---|
| `version` | `PROTOCOL_VERSION` (`1`) |
| `height` | `0` |
| `prev_hash` | 32 Null-Bytes (kein vorheriger Block) |
| `tx_root` | Merkle-Wurzel der leeren Liste |
| `timestamp` | `GENESIS_TIMESTAMP = 0` (rein strukturell) |
| `consensus` | die kanonischen Bytes der `network_id` |
| Transaktionen | leer |

Variante **Allokations-Genesis** (Mainnet-Kickstarter, M-next): das
Feld `consensus` trägt `network_id ‖ ALLOC-V1 ‖ count.v ‖ entries`
(vorbezahlte Domains/TLDs, kanonische Sortierung nach Name, Manifest
≤ 60 KiB, ≤ 2048 Allokationen) — dieses erweiterte Format
rechtfertigt `MAX_CONSENSUS_LEN` = 64 KiB (siehe
`scone-blockchain::genesis_alloc`).

**Verteilung der Allokations-Genesis**: kein
Genesis-Artefakt (`genesis.bin`, Manifest) lebt je im
Quell-Repository oder in Code-Releases, egal welches Netzwerk oder
welche Projektphase — zwei Genesis-Dateien sind zwei Netzwerke. Die
Mainnet-Allokations-Genesis entsteht nur zum Launch (die Umstellung
auf die standardmäßig aktivierte `mainnet`-Feature): erzeugt von
`scone-genesis` aus einem außerhalb des Repositorys
veröffentlichten signierten Manifest, ihr Hash wird in der
Launch-Ankündigung angeheftet, und nur Gründerknoten konsumieren
sie (`--genesis`, Geburtsstunde der Chain); alle anderen Knoten
stoßen per Sync zum Netzwerk. Das Werkzeug verweigert jede
Schreiboperation in ein Rust-Workspace-Verzeichnis (Anwesenheit von
`Cargo.toml`): eine Genesis darf nie im Quellbaum landen können.

Die `network_id` in der Konsensnutzlast macht die Genesis-Hashes —
und damit die gesamte Blockchain — **über Netzwerke hinweg
disjunkt**: ein Testnet-Block kann sich nie an eine Mainnet-Chain
anhängen (`UnknownParent`), kein Replay ist ohne Neu-Signieren
allesamt möglich.

API: `genesis_of(&NetworkParams) -> Block`,
`genesis_hash_of(NetworkId) -> BlockHash`; die Aliase `genesis()` /
`genesis_hash()` verweisen auf das Testnet (der
Entwicklungsstandard).

### Devnet — zurücksetzbares lokales Netzwerk

Das Devnet (`scone-devnet`) ist das dritte Netzwerk, gewidmet der
schnellen lokalen Iteration:

- **Eigene Genesis**: die Netzwerk-Id `scone-devnet` in der
  Konsensnutzlast ergibt einen Genesis-Hash, der von SOWOHL Testnet
  ALS AUCH Mainnet verschieden ist — keine Genesis, kein Block,
  keine Signatur und kein PoW-Proof überschreitet je die Grenze
  (Replay ist unmöglich).
- **Fast-null PoW**: Kurve im Bereich 0..=2 Bit angeheftet
  (`DEVNET_APEX = 2`, `DEVNET_FLOOR = 1` — der Längenzerfall gilt
  weiterhin): ein TLD- oder Domain-Claim mint in Millisekunden und
  durchläuft exakt denselben Codepfad wie die anderen Netzwerke.
- **Schneller PoS-Konsens**: dieselben Parameter wie das Testnet
  (Komitee von 4, 10-s-Epochen) für schnelle Finalität auf einem
  lokalen Mesh.
- **Zurücksetzen**: die Chain lebt in ihrem eigenen
  Datenverzeichnis (standardmäßig `~/.scone/devnet/`). **Das
  Löschen dieses Verzeichnisses und ein Neustart des Relays setzt
  die Chain zurück** (frische Genesis, jungfräulicher Store) — keine
  zusätzlichen Werkzeuge nötig. NIEMALS ein Devnet-Datenverzeichnis
  auf das eines anderen Netzwerks richten: ein nicht-leerer Store
  mit einer anderen Chain wird abgelehnt (`Corrupted`).

## Kanonischer Zustand

```text
DomainState  = { owner: OwnerId, sequence: u64, record_hash: Option<RecordHash>,
                 registered_at: u64, valid_until: u64 }
TldState     = enum { Private { owner: OwnerId } | Open }
               -- Enum (Owner-Option): die Open/Private-Invariante ist
               -- strukturell; die Todesfrist lebt außerhalb des Eintrags
               -- (tld_floors-Tabelle + journlierte Todesplanung)
ChainState   = NetworkParams                  (gebundenes Netzwerk, M8b)
               + DomainId -> DomainState      (direkter Zugriff, im Speicher)
               + TldId -> TldState            (TLD-Register, M7b)
               + DomainId -> (expired_at, OwnerId)   (Karenzfenster, M8b)
```

* `registered_at` / `valid_until` (M8b): Ablauf der Registrierung,
  ausgewertet gegen den **Zeitstempel des Parent-Blocks**
  (deterministisch, im Header committet — nie die lokale Uhr);
* `open`: eine private TLD kann **offen geboren** werden
  (`RegisterTld` mit `open = 0x01` — niemands, Self-Service) oder
  beim Claim **geschlossen** sein (geschlossen = nur assign), dann
  über `SetTldOpen` (Selbstregistrierung mit PoW) — **unumkehrbar** —
  offen werden;

- direkter Zugriff über `DomainId` (32 Bytes): nie ein `String` als
  Schlüssel, nie ein vollständiger Scan — eine notwendige
  Bedingung, um Hunderte Milliarden Domains anzustreben;
- `record_hash == None`, solange kein `UpdateDomain` angewandt
  wurde;
- die Id-Räume von TLD und Domain sind **konstruktiv disjunkt**
  (verschiedene Ableitungspräfixe, siehe
  `/docs/en/general/naming.md`): die Registrierung der TLD `uip`
  kann nie die Identität einer Domain besetzen, und umgekehrt.

### Anwendungsregeln

`state.apply(transaction)` ist deterministisch und atomar (bei
Fehler bleibt der Zustand unverändert):

**Alle Transaktionen (M8b, Wire 2026-09)**: Transaktionen tragen
kein `network`-Feld mehr — die Trennung beruht auf der
**Signatur**: die signierte Nutzlast injiziert die `network_id` der
Chain (`validate_transaction(tx, network_of_the_chain)`), also
scheitert eine für ein anderes Netzwerk signierte Tx an der
Signaturverifikation (`InvalidSignature`, VOR der Anwendung — eine
testnet-signierte Tx wird nie angewandt, nie von einem
Mainnet-Knoten gepoolt, und umgekehrt).

**REGISTER_TLD**:

- die TLD muss **frei** (im TLD-Register abwesend) und **nicht
  ICANN-reserviert** sein (`IcannTldReserved`: ICANN-Root-TLDs
  gehören auf Scone niemandem);
- **obligatorisches PoW**: `pow::verify` auf der Challenge
  `SCONE-TLD-V1 || tld` (abgeleitet aus dem von der Tx getragenen
  Namen), auf der **längenbasierten** Schwierigkeit des Labels
  (`pow_length`): private Geburt → `tld_difficulty`
  (Netzwerk-Apex), offene Geburt (`open = 0x01`) →
  `open_tld_difficulty` (Apex − 2) — der Digest wird von Grund auf
  neu berechnet (Salz `SCONE-POW-V1-ARGON2 ‖ network ‖ challenge ‖
  epoch`, Argon2id-Kern), die selbstdeklarierte Schwierigkeit muss
  der erwarteten gleichen, und die vom Proof **deklarierte Epoche**
  muss `height/POW_WINDOW` des anwendenden Blocks gleichen
  (Epochen-Salz gegen Pre-Mining — typisierte Zurückweisung
  `ProofEpochMismatch`, siehe `transactions.md`);
- bei Anwendung: `open = 0x01` → `{ Open }` (niemands —
  Self-Service, PoS-Sitz freigegeben, Todes-Untergrenze gelegt bei
  `now + Karenz`); `open = 0x00` → `{ Private { owner } }` —
  **standardmäßig geschlossen** (nur assign); das Öffnen erfolgt
  explizit über `SetTldOpen`.

**REGISTER (RegisterDomain)**:

- **die TLD des getragenen Namens muss registriert UND offen sein**:
  die Chain leitet `TldId(name.tld())` selbst ab (nie ein
  gelieferter Wert); abwesend → `UnknownTld` (D1, M7c), vorhanden
  aber geschlossen → `TldClosed` (M8b: der Nur-Assign-Pfad ist
  exklusiv);
- die Domain muss **frei** sein: im Zustand abwesend UND außerhalb
  jedes Karenzfensters einer abgelaufenen Registrierung (siehe GC
  unten);
- **obligatorisches PoW**: Challenge `SCONE-DOMAIN-V1 || name` auf
  der **längenbasierten Schwierigkeit des linksstehenden Labels**
  (`pow_length::domain_difficulty` — die TLD ist offen, das ist der
  Self-Service-Eintrittspreis);
- bei Anwendung: `{ owner, sequence: 0, record_hash: None,
  registered_at: now, valid_until: now + 1 Jahr }` — **die
  Registrierung dauert 1 Jahr** (365 Tage).

**Reihenfolge innerhalb des Blocks**: bedeutsam.
`[RegisterTld, SetTldOpen, RegisterDomain]` im selben Block ist
gültig; jede Permutation, die eine Regel anwendet, bevor ihre
Vorbedingung existiert, wird als ganzer Block abgelehnt (Undo-Log).

**UPDATE (UpdateDomain)**:

- die Domain muss existieren (nicht abgelaufen);
- `tx.owner` muss der aktuelle Owner sein;
- `tx.sequence` muss **genau** `current_sequence + 1` sein (keine
  Lücken, kein Replay);
- bei Anwendung: `sequence` rückt vor, `record_hash` wird ersetzt.

**TRANSFER_TLD (0x47)**:

- die TLD muss existieren und **privat** sein (sonst `UnknownTld` /
  `TldIsOpen` — eine offene TLD gehört niemandem);
- der Signer muss der aktuelle Owner sein (sonst `NotTldOwner`);
- bei Anwendung: `owner = new_owner` (die Identität des Empfängers
  ist opak; die Bindung an die Schlüssel, die sie ausgeben können,
  geschieht durch nachfolgende Txs). Keine Sequenz: die zuerst
  angewandte Übertragung gewinnt (Chain-Reihenfolge).

**REVOKE_TLD (0x6B)**:

- die TLD muss existieren und **privat** sein (eine offene TLD kann
  nicht widerrufen werden — `TldIsOpen`: sie stirbt natürlich in
  der GC), Signer = aktueller Owner;
- bei Anwendung: die TLD wird aus dem Register entfernt und durch
  ein frisches `RegisterTld` neu beanspruchbar (PoW
  eingeschlossen).

**SET_TLD_OPEN (0xB8)**:

- die TLD muss existieren und **privat** sein (bereits offen →
  `TldIsOpen`: der frühere Owner ist gelöscht, es existiert kein
  legitimer Schlüssel mehr, um dafür zu signieren); `open = 0x00`
  (Schließen) → typisierte Zurückweisung `TldCloseUnsupported` so
  früh wie die reine Validierung: **das Öffnen ist unumkehrbar**
  (Owner-Option);
- Signer = aktueller Owner (`NotTldOwner`);
- **Öffnen = obligatorisches PoW**: `pow::verify` auf der Challenge
  `SCONE-TLD-OPEN-V1 || tld_id` auf der flachen
  `open_tld_pow_difficulty` des Netzwerks (6 Bits Testnet /
  22 Bits Mainnet), Epochen-Salz eingeschlossen (das Proof reist im
  signierten Feld `open_proof` der Tx);
- bei Anwendung: `TldState::Open` — **der Owner wird gelöscht**
  (unumkehrbar, PoS-Sitz freigegeben), eine Todes-Untergrenze wird
  bei `now + Karenz` gelegt.

**ASSIGN_DOMAIN (0xD4)**:

- die TLD des getragenen Namens muss existieren und **privat** sein
  (sonst `TldIsOpen`: eine offene TLD ist nur Self-Service), Signer =
  Eigentümer der TLD (`NotTldOwner`);
- die Domain muss frei sein (gleiche Karenz-Regel wie REGISTER);
- bei Anwendung: `{ owner: assignee, sequence: 0, record_hash:
  None, registered_at: now, valid_until: now + 1 Jahr }`. **Kein
  PoW**: der Namespace-Eigentümer bürgt.

**TRANSFER_DOMAIN (0xE2, M8c)**:

- die Domain muss existieren (sonst `UnknownDomain`), Signer =
  aktueller Owner (`NotOwner`);
- bei Anwendung: `owner = new_owner` (der PoS-Sitz folgt der Domain
  in den Owner-Pool, symmetrisch zu `TransferTld`). Keine Sequenz:
  die zuerst angewandte Übertragung gewinnt (Chain-Reihenfolge).

**RENEW_DOMAIN (0x3C)**:

- die Domain muss existieren; **die TLD-Art wird ZUERST ausgewertet**
  (M8c-Härtung):
  - **private TLD**: **PoS**-Autorisierung — der Signer IST der
    TLD-Eigentümer, oder der Sitz des TLD-Eigentümers delegiert an
    den öffentlichen Schlüssel des Signers
    (`effective_operator(tld_owner) == renew.public_key` — die
    Delegation wird immer VOM TLD-Eigentümer gelesen, nie
    umgekehrt). Das Feld `proof` muss leer sein (ein nicht-leeres
    Proof = Zurückweisung).
  - **offene TLD**: **PoW**-Autorisierung — der Signer ist der
    aktuelle Domain-Owner (sonst `NotOwner`), und das Proof mint
    die Verlängerungs-Challenge: gleiche Schwierigkeit wie die
    Erstellung (nach Label-Länge), **gebunden an das aktuelle
    Ablaufdatum** (`challenge = DOMAIN_ID_VERSION ‖ kanonischer
    Name ‖ aktuelles valid_until`) — das Proof einer Verlängerung
    ist verschieden von dem der Registrierung und von jeder
    früheren Verlängerung (Anti-Wiederverwendung);
- `valid_until` muss die aktuelle Frist **strikt verlängern**
  (sonst `RenewalNotExtending`);
- `valid_until ≤ now + 3 Jahre` (`RenewalExceedsTerm` —
  Anti-Hording-Schranke: **1 Verlängerungsperiode = 1 Jahr,
  höchstens 3 Jahre im Voraus**);
- bei Anwendung: `valid_until = tx.valid_until`; die Verlängerung
  schiebt auch die Todes-Untergrenze der zugrunde liegenden offenen
  TLD auf (die Tx trägt den kanonischen Namen zu diesem Zweck).

**Deterministische GC (M8b)**: vor der Anwendung der Transaktionen
eines Blocks wertet die Chain die Abläufe zum **Zeitstempel des
Parent-Blocks** aus: jede Domain, deren `valid_until ≤ now`, wird aus
dem lebenden Zustand entfernt und in ein **30-Tage-Karenzfenster**
geparkt (`valid_until + 30 Tage`). Während der Karenz kann nur der
frühere Eigentümer den Namen neu beanspruchen; danach ist er für
alle frei. Die GC ist journliert: ein abgelehnter Block stellt den
Prä-GC-Zustand bit für bit wieder her.

**TLD-Lebenszyklus (feste Laufzeiten)**: TLDs tragen ihre eigene
Uhr, unabhängig von den Domains.

- **Private TLD**: der Claim eröffnet eine **3-Jahres-Laufzeit**
  (`TLD_TERM_SECS`). Die Verlängerung (`RenewTld`, 0x8F) bezahlt ein
  **PoW gleich der Erstellung** und setzt exakt 3 Jahre zurück
  (feste Laufzeit, bei Anwendung neu abgeleitet). Bei Ablauf öffnet
  sich eine **60-Tage-Karenz** (`TLD_GRACE_SECS`): die Verlängerung
  bleibt möglich, aber **jede andere Verwaltungsoperation ist
  eingefroren** — insbesondere `AssignDomain` (typisierte
  Zurückweisung `TldExpiredFrozen`): ein abgelaufener Eigentümer
  verliert die Verwaltungsrechte, bis er verlängert. Bereits
  zugewiesene Domains darunter leben ihr eigenes `valid_until` — die
  TLD friert ein, nicht ihre Kinder. Karenz ohne Verlängerung: die
  TLD stirbt in der GC (PoS-Sitz freigegeben, neu beanspruchbar).
  Ein `TransferTld` trägt die Restlaufzeit.
- **Offene TLD**: geboren (oder geöffnet) mit einer
  **1-Jahres-Untergrenze** (`OPEN_TLD_FLOOR_SECS` — gegen sofortiges
  Erstellen-Zerstören). Jede lebende Domain darunter schiebt die
  Todesuhr zurück (sie folgt dem Ablauf der jüngsten Domain). Keine
  Domain mehr + Untergrenze vergangen: Tod in der GC, neu
  beanspruchbar.
- **Unsterbliche offene TLD** (nur Genesis: `.scone`, `.tor` —
  verifizierbare Namensgebung über Onion-Adressen per CNAME —,
  `.p2p`): die Allokation `OpenImmortalTld` setzt den Sentinel
  `until = u64::MAX` — die GC kann ihn nie überschreiten, und das
  monotone Wieder-Auffrischen kann ihn nie senken: die TLD läuft
  nie ab, selbst wenn aller ihrer Domains entleert. Keine
  Laufzeit-Transaktion kann einen Unsterblichen erzeugen
  (`RegisterTld` behält seine 1-Jahres-Untergrenze); Domains
  darunter leben ihren normalen Zyklus.

Die TLD-Tode teilen die deterministische Todes-Warteschlange der
Domains (`tld_deaths`), journliert und bit-exakt wiederherstellbar.

**Ausnahme Höhe 1**: die Genesis trägt einen STRUKTURELL
nulloffenen Zeitstempel (`GENESIS_TIMESTAMP` = 0, eine
Protokollkonvention — keine Uhr). Die Chain-Zeit des ERSTEN Blocks
ist daher sein eigener, im Header committeter Zeitstempel
(deterministisch), nicht der seines Parents: die Verankerung der
Abläufe von Block 1 bei 0 würde die Todes-Untergrenze einer offen
geborenen TLD (und das `valid_until` einer bei Block 1 geborenen
Domain) auf die Unix-Epoche nageln — die Prä-Anwendungs-GC von
Block 2 (realer Zeitstempel) würde die Ressource vor der Anwendung
entfernen („dropped conflicting tx from block production: unknown
TLD“, Testnet-Regression 60).

**Replay**: Cross-Netzwerk-Replay ist auf drei Ebenen unmöglich —
disjunkte Geneseses (`UnknownParent`), `network_id` in jede
Signaturnutzlast injiziert (ungültige Signatur im anderen Netzwerk —
der einzige Ort, an dem das Netzwerk seit Wire 2026-09 erscheint),
PoW pro Netzwerk im Digest getrennt. Intra-Netzwerk-Replay einer
identischen Tx scheitert an Zustandsregeln (Doppel-Claim, Sequenz),
nie an der `TxId`.

Zwei Knoten desselben Netzwerks, die vom selben Zustand starten und
dieselben Blöcke in derselben Reihenfolge anwenden, erzeugen exakt
denselben Endzustand.

## Transaktionsreihenfolge

Die Reihenfolge ist eine grundlegende Eigenschaft, auf jeder Ebene
committet:

```text
Transaktionsreihenfolge im Block
    ↓
MerkleRoot
    ↓
BlockHash
    ↓
kanonische Chain
```

Zwei Transaktionen sind nicht austauschbar: im selben Block gilt
`[REGISTER, UPDATE]` angewandt, `[UPDATE, REGISTER]` wird abgelehnt
(Domain existiert zum Zeitpunkt des `UPDATE` nicht). Der globale
Tie-Break (der den Mempool ordnet, der den Block produziert)
gehört zum Konsens.

## Blockvalidierung (push_block)

Alles wird **neu berechnet**, nichts wird geglaubt — ein von einem
Peer geliefertes `tx_root`, Hash, Signatur oder Feld wird nie für
bare Münze genommen:

1. `prev_hash` == Hash der kanonischen Spitze (sonst
   `UnknownParent`, wenn der Parent unbekannt ist, `ParentNotTip`,
   wenn er bekannt, aber nicht die Spitze ist: minimale
   Fork-Erkennung);
2. `height` == Spitzhöhe + 1;
3. `version` == `PROTOCOL_VERSION` **exakt** (jede andere Version —
   niedriger oder höher — wird abgelehnt, nicht neu interpretiert);
4. Transaktionszahl ≤ `MAX_TXS_PER_BLOCK` (4096);
5. `tx_root` über die Transaktionen in Blockreihenfolge neu
   berechnet;
6. Konsens-Hooks (`validate_header`, `validate_tx`);
6ter. **fensterbasierte TXID-Anti-Replay (.bak-Port)**: eine
   Transaktion, deren `TxId` im Einschluss-Index der
   `REPLAY_WINDOW_BLOCKS = 256` jüngsten Blöcke erscheint, wird
   abgelehnt (`TxReplay`). Der Index (TxId → Einschlusshöhe) ist
   eine reine Funktion der kanonischen Chain, im selben Takt
   beschnitten; der Index wird beim Boot durch die Speicherung neu
   aufgebaut (Rescan der letzten `min(256, height)` persistierten
   Blöcke — M8, siehe `/docs/en/technical/storage.md`): ein neu
   gestarteter Knoten lehnt dieselben Replays ab wie ein
   laufender. Jenseits des Fensters bleibt die Wiedereinbindung den
   Zustandsregeln unterworfen (Doppel-Claim, Sequenz-Replay).
6bis. **signierter Produzent (M5, .bak-Port)**: die Nutzlast
   `consensus` muss als Produzentennutzlast dekodieren
   (`SCONE-BLOCK-V1 ‖ pk[32] ‖ sig[64]`, begrenzt); die Signatur
   deckt den *Produzenten-Signatur-Hash* = den `BlockHash` des
   Headers **mit geleertem Feld `consensus`** (die Nutzlast lebt in
   diesem Feld; den vollen Hash zu signieren wäre zirkulär); der
   Produzent muss zur beim `timestamp` des Blocks autorisierten
   Menge gehören (per Checkpoint gewähltes Komitee, plus
   freigeschaltete Recovery-Ziehungen; Bootstrap: Eigentümer
   lebender Domains — ein leerer Pool = offene Produktion, sonst
   wäre die erste Registrierung unmöglich). Die Genesis (Höhe 0,
   leere Nutzlast) ist strukturell und nie betroffen. Typisierter
   Fehler `InvalidProducer`;
7. für jede Transaktion, der Reihe nach: **kryptografische**
   Validierung (`validate_transaction`: Owner/Schlüssel-Bindung neu
   berechnet + `verify_strict` auf der von Grund auf neu
   berechneten signierten Nutzlast, siehe
   `/docs/en/technical/transactions.md`), dann Anwendung auf den
   kanonischen Zustand unter einem **Undo-Journal** (Undo-Log: ein
   Eintrag pro angewandter Transaktion, Kosten O(Transaktionen des
   Blocks), nicht O(Domains));
8. atomarer Commit: alles besteht ⇒ Block hinzugefügt, Journal
   verworfen; der geringste Fehler ⇒ Journal-Rückspulung (umgekehrte
   Reihenfolge), und die Chain bleibt strikt unverändert, bit für
   bit.

Keine Validierungsfunktion gerät in Panik; jede fehlerhafte oder
feindselige Eingabe erzeugt einen typisierten `BlockchainError`
(`OwnerKeyMismatch`, `InvalidSignature`, …).

## Blockzusammenstellung (BlockBuilder)

`BlockBuilder` (`scone-blockchain::builder`) setzt einen Block aus
einer Warteschlange validierter Transaktionen zusammen:

```text
BlockBuilder::after(parent_height, prev_hash)
    .with_timestamp(t) .with_consensus(payload)
    .push_tx(tx)… .build() -> Block
```

- `tx_root` über die geordnete Liste der Warteschlangen-Transaktionen
  **neu berechnet** (nie geliefert);
- `prev_hash`/`height` an den Parent gekettet (der Parent-Hash wird
  vom Aufrufer über `block_hash` berechnet, nie von einem Peer
  übernommen);
- `version` = `PROTOCOL_VERSION`;
- die Grenze `MAX_TXS_PER_BLOCK` wird beim Einfügen durchgesetzt
  (`TooManyTransactions`);
- die Nutzlast `consensus` (signierte Produzentennutzlast) und der
  `timestamp` werden vom Aufrufer geliefert: konkreter Konsens
  dockt über den bestehenden Trait
  [`Consensus`](#konsens-abstraktion) an, symmetrisch zur
  Validierung;
- M5: `.with_producer(&SigningKey)` signiert den Block als
  Produzent — die Nutzlast `consensus` wird zur signierten
  Produzentennutzlast (über den oben definierten
  Produzenten-Signatur-Hash) und ersetzt jedes frühere
  `.with_consensus`.

Ein zusammengesetzter Block besteht `push_block` ohne Neu-Kodieren
(getestet).

## Forks und Fork-Wahl (M3c, .bak-Port)

`push_block` bleibt der lineare Einstiegspunkt (Parent == Spitze,
sonst `UnknownParent`/`ParentNotTip`). `try_attach` (M3c) wertet
konkurrierende Blöcke aus:

- unbekannter Parent → `UnknownParent` (der Block wartet);
- gültige Abstammung, aber zurückliegend → Zweigwertung: die
  längste Chain gewinnt; perfektes Gleichstand → deterministischer
  Tie-Break (kleinster Spitz-Hash in lexikografischer Ordnung —
  zwei Knoten mit denselben Blöcken wählen immer denselben
  Gewinner, getestet);
- **Finalitätsuntergrenze**: ein Zweig, dessen einer Vorfahre einem
  finalisierten Checkpoint widerspricht (enthalten im
  `prev_checkpoint_hash` eines akzeptierten Checkpoints), wird
  abgelehnt (typisierter Fehler
  `Consensus("branch contradicts a finalized checkpoint")`);
- Adoption = Replay ab Genesis der Blöcke des gewinnenden Zweigs
  (bit-exakte Determinismus getestet: zwei Knoten, die dieselben
  Blöcke in verschiedener Reihenfolge erhalten, enden im selben
  Zustand);
- kein empfangener Block ist vor vollständiger Validierung
  kanonisch.

## Begrenztes RAM-Fenster (M7a, .bak-Port)

Die Chain hält nur die **letzten 512 Blöcke** im RAM
(`RAM_WINDOW_BLOCKS`, Port des `HISTORY_KEEP = 512` der .bak),
oberhalb der **Finalitätsuntergrenze** (der letzte finalisierte
Checkpoint):

- ein Block verlässt das RAM nur, wenn er **sowohl** außerhalb des
  Fensters liegt (`height < tip − 511`) **als auch** strikt unter
  der Finalitätsuntergrenze — ein Reorg über der Grenze bleibt
  immer möglich, diese Blöcke werden NIE verdrängt;
- vor dem ersten finalisierten Checkpoint (Bootstrap) wird
  **nichts verdrängt**: begrenzter Speicher beginnt mit der
  Finalität;
- die Verdrängung ist real: Blöcke, der Hash-Index (`known_hashes`,
  selbst begrenzt, nach Höhe indiziert) und Txs verlassen den
  Speicher — der Store (`NodeStore`, `scone-storage`) hält die
  dauerhaften Kopien;
- `block(h)` unterhalb des Fensters → `None`; `block_result(h)` →
  typisierter Fehler `BlockPruned { height }` — die Reorg-Pfade
  (`push_block`, `try_attach`, `adopt_branch`, Präfix-Replay)
  verbreiten ihn: das Relay muss das Segment aus der Speicherung
  neu laden oder auf einen vollen Sync zurückfallen (dokumentierte
  Einschränkung: keine segmentierte Adoption vor dem vollständigen
  Relay-Port);
- ein auf einem verdrängten Parent gebauter Block klassifiziert als
  `BlockPruned` (nicht `UnknownParent`): Parent-Höhe <
  `base_height` ⇒ der Store kann ihn liefern.

## Konsens-Abstraktion

```rust
pub trait Consensus {
    fn validate_header(&self, header: &BlockHeader) -> Result<()>;
    fn validate_tx(&self, tx: &Transaction) -> Result<()>;
}
```

- `PermissiveConsensus` (Standard): akzeptiert alles — verwendet,
  bis die echten Regeln definiert sind;
- die Hooks werden von `push_block` aufgerufen: konkreter Konsens
  (PoW, Schwierigkeit, Gebühren, Zeitstempel/Anti-Replay,
  Fork-Wahl) dockt an, ohne die Chain umzuschreiben;
- eine einzige Abstraktion, keine Trait-Hierarchie.

## Skalierbarer Zustand (KV-Backend, SMT-Pivot, Ablauf-Warteschlange, begrenzter Owner-Pool)

Ziel: 250+ Milliarden potenzielle Domains — NIEMALS alles in RAM
laden. Vier Bausteine:

### KV-Backend (`state_backend.rs`)

```rust
pub trait StateBackend: Default + Send {
    fn get(&self, key: &Pkey) -> Option<Vec<u8>>;
    fn put(&mut self, key: Pkey, value: Vec<u8>);
    fn delete(&mut self, key: &Pkey);
    fn range(&self, from: Option<&Pkey>, limit: usize)
        -> (Vec<(Pkey, Vec<u8>)>, Option<Pkey>);
}
```

- `Pkey` = die 32 rohen Bytes der `DomainId`/`TldId` — EIN
  geordneter Schlüsselraum (disjunkte Ableitungspräfixe), Werte =
  die Kodierungen `SCONE-ENTRY-DOM-V1` (88 feste Bytes) /
  `SCONE-ENTRY-TLD-V1` (variabel: 33 Bytes privat / 1 Byte offen,
  Owner-Option-Enum — Flag + bedingter Owner, kein Feld `until`
  mehr);
- `MemoryBackend` (BTreeMap) = Referenzimplementierung (Dev,
  Tests); KEINE konkrete Abhängigkeit in `scone-blockchain` (redb
  im Crate verboten) — Protokoll/Konsens hängen allein vom TRAIT
  ab;
- `CachedBackend<B>` (`state_cache.rs`) = begrenzter LRU-Cache über
  einem inneren Backend: Write-Through (das innere ist immer
  aktuell), Misses nie gecacht, `range` blättert das innere —
  dieselben Antworten wie das innere Backend allein
  (Determinismusvertrag, getestet);
- `RedbBackend` (`scone-storage::redb_backend`) = persistiertes
  Backend: Tabelle `state_kv` (32-Byte-Rohschlüssel, lexikografische
  Ordnung = erforderliche aufsteigende Ordnung), Zähler in der
  Meta-Tabelle `state_kv_meta` gepflegt (nie COUNT);
  `CachedBackend<RedbBackend>` = Zustand mit begrenztem
  RAM-Fußabdruck — die „leichte Stufe“;
- `smt_persist`: konsumiert die Dirty-Sets des SMT
  (`take_dirty_sets`) und schreibt sie unter reservierten
  Schlüsselpräfixen in ein Backend (0x01 Knoten, 0x02 Blatt) —
  Fundament des seitenweisen SMT;
- der Trait verlangt kein `Default` mehr (eine
  Festplatten-Datenbank hat keinen sinnvollen Standardwert);
- `range` iteriert in aufsteigender Schlüsselreihenfolge
  (Determinismusvertrag) — nur Archivpfad, nie Konsensregel;
- `ChainState` greift auf Domains/TLDs NUR über den Schlüssel durch
  das Backend zu (Punkt-Lesevorgänge, journlierte Schreibvorgänge).

### Kanonische SMT-Wurzel (Pivot)

- `state_root_smt()` (O(1), gecachte Wurzel, `SCONE-STATE-SMT-V1`)
  ist die **kanonische** state_root: sie ist es, was
  `checkpoint_data` committet und was `accept_checkpoint` an der
  Spitze verifiziert. Die Berechnung durchläuft NIE die Menge der
  Domains;
- die alte direkte Faltung `SCONE-STATE-V1` überlebt als
  `state_root_archive()`: O(N), Archiv-/Verifikationspfad (volle
  Neu-Faltung aus der Speicherung, um eine historische Wurzel
  kreuzzuprüfen) — kein Konsens-Artefakt. Beide committen
  dieselben Blätter: gleiche logische Zustände ⇒ gleiche Wurzeln
  (für beide);
- ein akzeptierter Checkpoint mit der SMT-Wurzel verifiziert nach
  einer Wiederherstellung (Reload) neu: die Wurzel ist eine reine
  Funktion des wiederhergestellten Zustands (Backend + SMT +
  Index).

### Ablauf-Warteschlange (GC O(abgelaufen), nicht O(N))

- `expirations: BTreeMap<u64, BTreeSet<DomainId>>` —
  Ablaufzeitpunkt → dann ablaufende Ids (Ids pro Slot sortiert:
  Zustandsgleichheit und Abfluss-Reihenfolge unabhängig von der
  Ankunftsreihenfolge);
- `gc_expired_journaled(now)` entleert `range(..=now)` statt den
  ganzen Zustand zu scannen: Kosten O(zu diesem Zeitpunkt
  Abgelaufene);
- RAM-Schranke: ein Eintrag pro LEBENDEN Domain (nur ausstehende
  Abläufe — eine Verlängerung verschlüsselt den Slot neu, die GC
  entleert ihn), nie ein Knoten pro Domain in einer
  Scan-Struktur;
- vom Undo-Journal gepflegt (bit-exaktes Rollback: der Slot wird
  wieder eingesetzt).

### Begrenzter Owner-Pool (Schlüssel, nicht Domains)

- `owner_refcounts: HashMap<OwnerId, u64>` — ein Zähler pro
  VERSCHIEDENEM lebenden Owner (Domänen + TLDs), vom Journal bei
  jedem register/transfer/revoke/assign/GC gepflegt
  (acquire/release);
- `eligible_validators(now)` liest diesen Index — O(Owners), NIE
  eine Iteration über Domains. Zeitliche Lebendigkeit ist bereits
  garantiert: die deterministische GC (Parent-Zeitstempel) hat jede
  abgelaufene Domain UND ihren Refcount vor dem Aufruf entfernt.
  Sortierte Ausgabe (deterministisch);
- RAM-Schranke: O(verschiedene Owners), unabhängig von der Anzahl
  der Domains;
- `owner_pool_root` (Archiv-Faltung) faltet diese sortierten Owners —
  ebenfalls kein Domain-Scan.

RAM-Budget im Dauerbetrieb (Festplatten-/Netzwerk-Backend):
O(verschiedene Owners + ausstehende Abläufe + laufende
Karenzfenster + Undo-Journal) — alle unabhängig von der Anzahl der
Domains begrenzt, alle bit-exakt vom Journal wiederhergestellt.

## Sitze und das Finalitätskomitee (exakter PoS-Mechanismus)

Dieser Abschnitt beschreibt den EXAKTEN Berechtigungs- und
Komiteewahl-Mechanismus wie implementiert (`finality.rs`) — das
operative Komplement (Anker-Schleife, Wire) lebt in
`/docs/en/technical/relay.md`, die Sicherheitsanalyse in
`/docs/en/technical/security-model.md`.

### Ein Sitz = Besitz (keine Registrierung)

Ein PoS-**Sitz** ist KEINE Registrierung: er ist der Besitz einer
lebenden Domain oder TLD. `eligible_validators(now)` berechnet den
Pool:

- Schlüssel der lebenden Owners über den begrenzten Index
  `owner_refcounts` (ein VERSCHIEDENER Owner pro lebender
  Domain/TLD — hundert Domains unter einem Schlüssel = EIN Sitz:
  das wirtschaftliche Anti-Sybil ist pro Identität, linear);
- eine TLD zählt (ihr Claim kostete ein PoW — der Eigentümer einer
  lebenden TLD ist ein Stakeholder), ebenso eine lebende Domain
  (`valid_until > now` — garantiert durch die deterministische GC,
  bereits vor dem Aufruf entleert);
- **abzüglich verbannter Schlüssel** (M9: ein ge-slashter
  Äquivokant ist lebenslang aus dem Pool);
- eine **offene** TLD (`owner: None`) hält KEINEN Sitz: ihr
  früherer Eigentümer wird beim Öffnen freigegeben (`owner_release`
  im `SetTldOpen`-Apply), und sie stirbt in der GC, ohne je wieder
  abzustimmen.

Sortierte Ausgabe (deterministische Reihenfolge, unabhängig von der
Iteratorenreihenfolge des Index).

Strukturelle Folge: **der Tod des Sitzes = der Tod der
Domain/TLD**. Ablauf + GC, `RevokeTld` (Sitz freigegeben),
`TransferTld`/`TransferDomain` (der Sitz folgt der Domain zum neuen
Owner), `SetTldOpen`-Öffnung (Sitz freigegeben) — jeder
Eigentumsübergang bewegt oder gibt den Sitz im Pool frei,
symmetrisch zum `owner_refcounts`-Journal (acquire/release).

### Pool EINGEFROREN beim letzten finalisierten Checkpoint (`FinalizedBase.pool`)

Bei Annahme eines Checkpoints (`accept_checkpoint`) erfasst die
Finalitätsbasis den Pool des Zustands BEIM FINALISIERTEN BLOCK:

```text
FinalizedBase { checkpoint, pool, block_ts, seed }
```

- `pool` = `eligible_validators(Zeitstempel des finalisierten
  Blocks)` — die Finalität liest NIE wieder den lebenden Zustand:
  jede Registrierung (Claim, Transfer, Bann) nach dem Checkpoint
  zählt nur für spätere Epochen;
- `seed` = `next_seed(signing_hash(checkpoint))` — der Samen der
  NÄCHSTEN Epoche leitet sich vom **Inhalts**-Hash des Checkpoints
  ab (keyed, Signaturmenge ausgeschlossen — zwei Sichten mit
  verschiedenen Quorum-Teilmengen desselben Checkpoints wählen
  dasselbe Komitee).

### Komitee-Ziehung (Bootstrap, Frische, Degradation)

`committee(recovery)`:

1. **Bootstrap** (kein finalisierter Checkpoint): die **FRISCHEN
   Betreiber** des Pools signieren (`fresh_operators` — ein
   recenter Heartbeat, dasselbe Fenster wie das finalisierte
   Regime). Der Filter ist Chain-Zustand (die Heartbeats sind
   On-Chain), also deterministisch gemäß der Spitze: zwei Knoten
   an derselben Spitze berechnen dieselbe Menge. Unter
   `MIN_FINALITY_COMMITTEE_SIZE` frischen Betreibern sicherer
   Rückfall auf den GESAMTEN Pool (kalter Bootstrap, allgemeiner
   Betreiber-Ausfall — die Finalität geht konstruktiv nie
   verloren). Der rohe Pool kann nicht als Signiermenge dienen:
   jede Self-Service-Registrierung wächst den
   Quorum-Nenner, während diese Eigentümer nie signieren — die
   Finalität wurde nach ~10 Registrierungen unerreichbar
   (Testnet, 2026-10-21).
2. **Finalisiertes Regime**: Ziehung aus der **frischen** Teilmenge
   des eingefrorenen Pools (RFC Proof-of-Service §5.2) — jeder Sitz
   (Owner-Schlüssel) wird zum EFFEKTIVEN Betreiber durch den
   Zustand an der Spitze aufgelöst
   (`SetTldOperator`/`SetDomainOperator`-Delegation = ein
   recenter On-Chain-Fakt, oder der Owner-Schlüssel bei
   Selbstbetrieb); der Sitz nimmt UNTER SEINEM EFFEKTIVEN
   SCHLÜSSEL teil (das ist der, der signiert), sofern er einen
   **frischen Heartbeat** hat (gleitendes Fenster). Verbannte
   Schlüssel bleiben ausgeschlossen.
3. **Sichere Degradation**: weniger als
   `MIN_FINALITY_COMMITTEE_SIZE` (4) frische Betreiber → Ziehung
   aus dem GESAMTEN Pool (kalter Bootstrap, allgemeiner
   Betreiber-Ausfall — die Finalität geht konstruktiv nie
   verloren).

`allowed_producers(block_ts)` (Blockproduktion) folgt einer
bewusst weiteren Regel: gesamter Pool beim Bootstrap; **gesamter
EINGEFRORENER Pool** im finalisierten Regime. Das gewählte Komitee
(+ Recovery-Ziehungen) bleibt nur die CHECKPOINT-SIGNIERMENGE. Die
Verengung der Produktion auf das Komitee wurde zuerst spezifiziert
und implementiert, dann wieder entfernt: ein Produzent, dessen Sicht
um einen Checkpoint zurückliegt, bewertet seine Kandidaten nach den
Bootstrap-Regeln und emittiert Blöcke, die jeder finalisierte Peer
ablehnt (`InvalidProducer`), und jede über den Übergang
zusammengesetzte Chain-Historie kann dann nie mehr das
`pool_at_height`-Replay passieren — die Finalität friert am ersten
Checkpoint für immer ein (Sim-Reproduzent `duplicate_and_delay`,
2026-10-21). Allein der Pool-Freeze trägt die Sicherheitslast:
Mitgliedschaftsänderungen (Registrierung, Transfer, Bann, Ablauf)
werden erst am NÄCHSTEN Checkpoint wirksam, sodass die
Produzentenmenge eine reine Funktion der finalisierten Chain bleibt.

Das Checkpoint-Quorum ist `quorum_for(Komiteegröße)`
(floor(2n/3)+1); ein Komitee unter `MIN_FINALITY_COMMITTEE_SIZE`
finalisiert nie (BFT-Wächter).

## An Konsens zurückgestellt

Bewusst in diesem Crate undefiniert:

- **Mempool**: feinkörnige Zulassung/Sortierung (Gebühren,
  Ersetzung) bleibt im Relay (M4 brachte Cap pro Domain +
  fensterbasierte TXID-Anti-Replay — `REPLAY_WINDOW_BLOCKS = 256`);
- **globale Ordnung**: die Mempool-Sortierung ist eine lokale
  deterministische Sortierung (Relay), keine globale Autorität;
- **Zeitstempel-Autorität** jenseits der GC: Abläufe (M8b) nutzen
  den Zeitstempel des Parent-Blocks; andere Zeitregeln (feinkörniges
  Anti-Replay) bleiben im Konsens;
- verteilte Speicherung: der Zustand ist im Speicher; das Backend
  (`scone-storage`, redb) wendet diese Regeln an, ohne sie zu
  duplizieren.

Geliefert seit M8b (zuvor zurückgestellt): Registrierungs-PoW pro
Netzwerk (`RegisterTld` immer, `RegisterDomain` auf offener TLD),
Ablauf/Verlängerung 1 Jahr + 30-Tage-Karenz + 3-Jahres-Horizont,
deterministische GC zum Parent-Block-Zeitstempel,
Cross-Netzwerk-Replay unmöglich (Genesis + signiertes Feld +
PoW-Digest).

## Deterministische verteilte Simulation (P1.6)

`crates/scone-blockchain/src/sim/` enthält ein **deterministisches**
Netzwerk-Simulations-Gerüst (`SimNet`): N echte `Blockchain`s (keine
Mocks), angetrieben von einer **virtuellen Uhr** und einer
**Ereignis-Warteschlange**, priorisiert nach `(Tick,
Einfügereihenfolge)`. Keine Sockets, kein async, keine
Wanduhr — jede Ziehung (Verlust, Latenz, Duplikation) ist eine reine
Funktion von `(Samen, Sende-Nummer)`, sodass ein Szenario mit
festem Samen byte für byte reproduzierbar ist.

### Modell

- **Transport**: jede Aussendung erfährt wahrscheinlichen Verlust
  (konfigurierbar in ‰), künstliche begrenzte Latenz
  (`latency_ticks`), mögliche Duplikation (eine verzögerte Kopie)
  und die aktuelle Partitions-Karte (Nachrichten zwischen Gruppen
  gehen verloren). Duplikationen treiben die logische Uhr nie voran
  und lösen keine neuen Sendungen aus: der reale Ereignisstrom ist
  mit oder ohne Duplikation identisch (Determinismusvertrag,
  bewiesen durch den Test `duplicate_and_delay`).
- **Epidemisches Gossip**: ein Knoten sendet einen
  Block/eine Transaktion/einen Checkpoint an seine verbundenen
  Peers (gleiche Partition) nur beim ersten Anblick — Relay-Regel
  H2 (schneidet Schleifen). Ein geparkter (Waisen-)Block wird NIE
  neu gesendet; Empfangslücken werden durch den
  Synchronisierungspfad (`GetBlocks`) gefüllt, wie im Relay.
- **Produktion**: ein globaler Produktionsschritt pro Intervall;
  jede Partitiongruppe produziert höchstens EINEN Block pro Runde,
  durch ihren autorisierten Knoten mit der besten Höhe. Während des
  Bootstraps (vor der ersten Finalität) ist der Produzent auf den
  niedrigsten Knoten angeheftet: die autorisierte
  Produzentenmenge leitet sich aus dem Zustand ab (Pool lebender
  Domains), zwei leicht verschiedene Sichten wählen verschiedene
  Komitees, und ein lokal legitimer Block wäre woanders
  `InvalidProducer` — eine dauerhafte Divergenz. Nach der ersten
  Finalität ist das Komitee in der Finalitätsbasis eingefroren
  (identisch auf jedem konvergierten Knoten), und die Produktion
  ist frei.
- **Finalität**: bei jedem Vielfachen von `sign_interval` Höhen
  signiert der designierte Anker des Komitees den deterministischen
  Inhalt seiner Spitze über einen echten absturzsicheren
  `SignerGuard` (ein Schlüssel = eine Stimme pro Epoche, Zustand
  vor der Diffusion fsynced); die anderen Komiteemitglieder
  signieren bei Empfang (nach Prüfung, dass der referenzierte Block
  tatsächlich ihr kanonischer Block auf dieser Höhe ist — Wächter
  auf Relay-Ebene) und senden das angereicherte Aggregat erneut
  aus. Bei Quorum finalisiert `accept_checkpoint`. Jeder Knoten
  committet auf den ersten gesehenen Vorschlag pro Epoche
  (Bootstrap-Wächter gegen Doppel-Finalität: zwei konkurrierende
  Vorschläge derselben Epoche mit disjunkten Signern können nicht
  beide akkumulieren).
- **Absturz/Neustart**: ein abgestürzter Knoten verliert seinen
  gesamten RAM-Zustand (Chain, Mempool, Parking, Sichten), behält
  aber sein Signer-Guard-Verzeichnis auf der Festplatte, exakt wie
  ein echter Prozess; er resynchronisiert sich durch das Abspielen
  der kanonischen Blöcke, die seine Peers liefern
  (`GetBlocks`/`GetCheckpoints`).

### Abgedeckte Szenarien (`sim::tests`)

| Test | Was bewiesen wird |
|---|---|
| `convergence_simple` | 8 Knoten, 5 ‰ Verlust, Latenz 1–5 Ticks, 500 Blöcke: dieselbe Spitze, derselbe `state_root_smt` überall; aktive Finalität; nie zwei inkompatible Checkpoints pro Epoche. |
| `partition_then_heal` | Partition 4a/4b für ~300 Blöcke: beide Seiten produzieren, Heilung → Konvergenz mit Reorg des verlierenden Zweigs; gekreuzte Checkpoint-Fenster widersprechen nie einer geteilten Epoche (vor und nach der Heilung). |
| `crash_restart` | Absturz bei h≈200, Neustart bei h≈400: Resynchronisation durch Abspielen der Netzwerk-Blöcke — dieselbe Höhe, Spitze, Zustand; die Transaktion nach dem Absturz ist für den auferstandenen Knoten sichtbar. |
| `duplicate_and_delay` | 10 ‰ Duplikation: Endhöhe, Zustandswurzel und Anzahl finalisierter Checkpoints identisch zum Lauf ohne Duplikation (Idempotenz: TxReplay/Seen-Sets absorbieren Duplikate). |
| `concurrent_tx` | Zwei konkurrierende Owners auf derselben Domain: genau ein Update gewinnt (der Owner — `NotOwner` für den Eindringling, saubere typisierte Zurückweisung), nie eine doppelte Zustandsanwendung, die verlierende Tx gelangt in keine Chain. |

### Parameter und Dauer-Schranken

Standardwerte: `produce_interval` 7 Ticks, `sign_interval` 16
Höhen, `sync_interval` 29–60 Ticks (standardmäßig 41, in einigen
Tests gesenkt), `Horizont` 200k–500k virtuelle Ticks. Jeder Test
bettenet einen Wanduhr-Wächter < 60 s im Debug-Modus ein
(`assert_under_60s`). Die dominierenden Kosten sind der
Validierungskern (~9 ms pro leerem `push_block_with_gc` im Debug,
Ed25519 unoptimiert): Szenarien bleiben unter ~8 Knoten × 500
Blöcke.

### Grenzen

- **Keine echte Transportschicht**: dieses Gerüst übt die
  Chain-Logik unter widriger Zustellung, nicht den Schwarm (libp2p,
  Request-Response, Backpressure, Erkennung). Die echten
  E2E-Tests (`crates/scone-network/tests/`) bleiben der
  Transport-Nachweis.
- **Batch-Synchronisation im RAM**: gelieferte Blöcke stammen aus
  dem RAM-Fenster (Szenarien bleiben unter `RAM_WINDOW_BLOCKS`);
  der `BlockPruned`- + Store-Neulade-Pfad wird nicht simuliert (von
  den Unit-Tests der Chain abgedeckt).
- **Ehrliches Anker-Modell**: Anker signieren nur gegen ihre eigene
  kanonische Chain verifizierbare Inhalte; byzantinische
  Äquivokation (absichtliches Doppelt-Signieren) wird durch den
  `SignerGuard` verhindert, aber nicht als aktiver Angriff
  ausgeübt.
- **Eine Partition gleichzeitig**: die Gruppenkarte ist global und
  wird sequenziell vom Treiber gewechselt; kontinuierlich
  überlappende/entwickelnde Partitionen werden nicht modelliert.

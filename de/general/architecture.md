# Architektur

## Zuerst in einfachen Worten

Fünf Konzepte genügen, um alles andere zu verstehen:

1. **Ein Name** — eine Adresse im Internet, wie `mysite.scone`.
2. **Ihre 24-Wörter-Phrase** — Ihre Identität. Sie ist die Urkunde,
   die beweist, dass ein Name Ihnen gehört, und signiert jede
   Änderung, die Sie vornehmen.
3. **Das öffentliche Registerbuch** (die Blockchain) — das
   gemeinsame, öffentliche Hauptbuch darüber, wem was gehört. Niemand
   kontrolliert es, niemand kann es umschreiben.
4. **Mining** — so reservieren Sie einen Namen: Ihr Computer verrichtet
   Arbeit, fair und gleich für alle. Keine Warteschlange, kein
   Mittelsmann.
5. **Ein Relay** — ein Computer, der das Netzwerk betreibt, indem es
   eine Kopie des Buches führt und Fragen beantwortet. Jeder kann
   eines betreiben; je mehr es gibt, desto stärker das Netzwerk.

Der Rest dieser Seite geht tiefer — Sie können hier aufhören und
verstehen das Projekt bereits.

## Leitprinzip

Scone trennt Autorität von Verfügbarkeit:

- die **Blockchain** ist die Quelle der Wahrheit für Eigentum und die
  Reihenfolge der Änderungen;
- der **DHT** ist ein Verteilungs-/Replikationssystem, er ist
  **keine** Autorität;
- die **lokale Speicherung** hält den Zustand des Knotens, Indizes
  und Caches.

Ein Knoten muss alle aus dem DHT stammenden Daten als potenziell
bösartig behandeln: sie werden nur nach kryptografischer Prüfung
akzeptiert (Owner-Signatur, Abgleich mit Ownership / sequence /
record_hash der Chain).

## Trennung der Verantwortlichkeiten

```text
BLOCKCHAIN
──────────
Eigentum (Ownership)
Registrierung
Autorisierung
Reihenfolge
Sequenz (Sequence)
Eintrags-Hash (Record hash)
Konsensinformationen

DHT
───
DNS-Einträge
Signierte Einträge
Replikation
Verfügbarkeit

LOKALE SPEICHERUNG
──────────────────
Lokaler Zustand
Indizes
Gecachte DHT-Einträge
Blockchain-Zustand
```

## Crates und Abhängigkeiten

```text
scone-crypto     ← unterste Schicht: reine Primitive über Bytes
    ▲
scone-core       ← Protokolltypen (hängt von scone-crypto ab)
    ▲
scone-protocol  ← Binärformat Wire/Blockchain
    ▲
scone-blockchain← Blockchain-Logik, kanonischer Zustand (im Speicher)
    ▲               └ scone-topology (reine Engine, null Netzwerk/FS/
                     async; gespeist von scone-network) — scone-genesis
                     (einmaliges Mainnet-Genesis-Werkzeug-Binary)
scone-storage   ← lokale Persistenz: redb-Backend + verifizierte Snapshots
    ▲
scone-network   ← p2p-Relay: libp2p/QUIC, Mempool, Sync, DHT, DNS
    ▲
scone-keystore  ← verschlüsselte `.sconekey`-Schlüsseldateien (hängt von scone-crypto ab)
    ▲
scone           ← Binärdatei / Anwendung
```

### Entscheidung: `scone-core` hängt von `scone-crypto` ab

Die `DomainId` muss von der Grundlage her berechenbar sein
(`BLAKE3-256(prefix || kanonischer Name)`), aber konkrete Kryptografie
darf den Kern nicht kontaminieren. Lösung:

- `scone-crypto` ist die unterste Schicht: rein deterministische
  Funktionen über Bytes (plus der OS-Zufallsgenerator für die
  Schlüsselerzeugung), null Core-Typen, keine Netzwerk-/Dateisystem-/async-Abhängigkeiten → keine Zirkularität möglich;
- `scone-core` nutzt es ausschließlich für Hashing;
- Ed25519-Signaturen und Schlüssel leben in `scone-crypto` und
  arbeiten auf Bytes (kanonische Nutzlasten, erzeugt von
  `scone-protocol`).

### Rolle jedes Crates

**scone-core** — Namen (`TldName`, `DomainName`), Bezeichner
(`DomainId`, `OwnerId`, `PublicKeyRef`), DNS-Daten (`DnsRecord`,
`SignedDnsRecord`), Transaktionen (`RegisterDomain`, `UpdateDomain`).
Deterministisch und portabel. In diesem Crate verboten: libp2p, redb,
tokio/async, Netzwerk, Dateien, OS, DNS-Server, Blockchain-Logik.

**scone-crypto** — kryptografische Primitive. Heute: `hash256`
(BLAKE3), Ed25519-Signaturen und Schlüssel (`SigningKey`,
`PublicKey`, `Signature`, strenge Stapelverifikation), AEAD-Hülle
von Eintragsgruppen (XChaCha20-Poly1305). Geplant: Merkle-Baum.
Regel: keine Eigenbau-Algorithmen, nur geprüfte Abhängigkeiten.

**scone-protocol** — Binärformat des Protokolls: kanonische
Kodierung/Dekodierung (minimales LEB128-Varint, schlanke 32-Byte-Bezeichner)
von Transaktionen, DNS-Einträgen, Blöcken (`Block`/`BlockHeader` mit einem
opaken Konsensfeld) und P2P-Nachrichten, mit Größenbegrenzungen, die
vor jeder Allokation geprüft werden, Zurückweisung nicht-kanonischer
Formen und Versionierung (`PROTOCOL_VERSION`). Keine Konsens-,
Netzwerk- oder Speicherlogik.
Format-Referenz: `/docs/en/technical/protocol.md`.

**scone-blockchain** — Blockchain-Logik und kanonischer Zustand:
Transaktionsidentifikation (`TxId`), geordneter Merkle-Baum,
`BlockHash`, deterministische Genesis, Block- und Transaktionsvalidierung,
Zustandsanwendungsregeln (`DomainState`/`ChainState`, im Speicher,
direkter Zugriff über `DomainId`), Fork-Erkennung und Zweigwahl
(`fork_choice`: Finalitätsuntergrenze + längste Chain + niedrigster
Hash als Tie-Break), Checkpoint-Finalität und Konsens-Abstraktion
(`trait Consensus`, standardmäßig permissive Implementierung). Das
Registrierungs-PoW (minen/verifizieren, schwieirgkeitsgrade pro Netzwerk) lebt in
`scone-core`. In diesem Crate verboten: Netzwerk, DHT, Speicherung (redb),
DNS-Server. Referenz: `/docs/en/technical/blockchain.md`.

**scone-storage** — lokale Persistenz (ausgeliefert). Das konkrete Backend ist
`redb` (`NodeStore`), verborgen hinter einem minimalen Trait, damit
Blockchain/DHT/DNS nie direkt an redb gekoppelt sind; ein anderes
Backend (verteiltes KV) wäre anschließbar. Enthält das paginierte
KV-Zustands-Backend, das Block-Log, verifizierte Snapshots (gehashtes Manifest
+ Seiten) und den Wiederherstellungs-/Replay-Pfad beim Neustart. Referenz:
`/docs/en/technical/storage.md`.

**scone-network** — das p2p-Relay des Knotens: libp2p/QUIC-Schwarm,
Transaktions- und Block-Relay (systematische lokale Validierung),
begrenzter Mempool, begrenzte Block-Synchronisierung (128 Blöcke/Anfrage),
Kademlia-DHT für `SignedDnsRecord`, UDP+TCP-DNS-Server (GeoDNS immer
aktiv — eingebettete GeoIP-Tabelle, rekursiver Fallback), die
Checkpoint-/Heartbeat-Schleife der Anker (die CLI spricht direktes
p2p-QUIC mit dem Relay, kein lokales RPC mehr). Erreichbarkeit und
Erkennung: AutoNAT v2 (Urteil erreichbar/NATed), Circuit-Relay v2 +
DCUtR für NATed Knoten, mDNS (Null-Konfigurations-Erkennung im LAN,
auf dem Devnet standardmäßig AN) und UPnP (QUIC-Portfreigabe durch
den Router, Opt-in). Konfiguration über eine `.env`-Datei (siehe `.env.example`).
Referenz: `/docs/en/technical/relay.md`.

**scone-keystore** — verschlüsselte Speicherung von Ed25519-Signaturschlüsseln
(`.sconekey`-Dateien: Argon2id + XChaCha20-Poly1305). Dieses Crate
besitzt das Dateisystem, damit `scone-crypto` rein bleiben kann. Referenz:
`/docs/en/technical/keystore.md`.

**scone-topology** — REINE Engine zur lokalen p2p-Topologieoptimierung:
null Netzwerk, null Dateisystem, null async. Sie kennt weder libp2p
noch den Schwarm: `scone-network` liefert ihr Beobachtungen (Ping-RTT,
Verbindungen) und erhält Entscheidungen zur Nachbarschaftsverwaltung.

**scone-genesis** — einmaliges Werkzeug-Binary (Kickstarter) zur
Vorbereitung der Mainnet-Genesis: versioniertes JSON-Manifest,
Allokationen, deterministischer Block.

**scone** — Binärdatei/CLI. Hängt nur von einem Crate ab, wenn es ihn
tatsächlich nutzt (heute: scone-network für das Relay, scone-blockchain,
scone-storage, scone-keystore, scone-protocol, scone-core,
scone-crypto, scone-topology).

## Der Knoten

Ein Scone-Knoten vereint:

1. eine Blockchain (Konsensvalidierung, Eigentumszustand);
2. DHT-Routing (Veröffentlichung/Abruf von `SignedDnsRecord`);
3. lokale Speicherung (Zustand, Indizes, Caches);
4. einen Resolver / DNS-Server an der Front.

### Start eines neuen Knotens (Zustands-Bootstrap, P0.3/P0.4)

Ein neuer Knoten spielt die Genesis NICHT ab: er startet aus einem
verifizierten finalisierten Zustand.

```text
finalisierter Checkpoint (Komitee, Signatur-Quorum — validiert von
        │              der Chain-Schicht vor jeder Nutzung)
        ▼
verifizierter Snapshot (Manifest + paginierte Seiten; der Importeur
        │              BERECHNET die SMT-Wurzel aus den Einträgen
        │              neu und gleicht sie mit der signierten
        │              state_root ab — ein gefälschter/unvollständiger
        │              Snapshot wird vor jedem Schreiben abgelehnt)
        ▼
Ankerblock H (vom Netzwerk NACH HASH angefordert)
        │
        ▼
jüngste Blöcke H+1..H+k (Relay, steigende Höhe)
        │
        ▼
synchronisiert — beim Boot wird NUR das Suffix über H abgespielt
```

Zwei Knotenprofile:

- **Archivknoten** (Betreiber-Flag): behält alle historischen Blöcke,
  stellt sie dem Netzwerk bereit, startet über die bestehenden Pfade;
- **Vollzustands-Knoten**: besitzt den vollständigen Zustand bei H und
  das Suffix, aber nicht die Historie unter H — er validiert
  zukünftige Blöcke und stellt Zustand bereit, keine Historie.

Dieser Pfad ist OPT-IN: der Import eines verifizierten Snapshots ist
eine explizite Aktion des Betreibers (`import_verified_snapshot`),
nie automatisch — das Standardverhalten eines bestehenden Knotens
ändert sich nicht.
Normative Details: `/docs/en/technical/storage.md`, Abschnitte
„VERIFIZIERTER Zustands-Snapshot“ und „Zustands-Bootstrap“.

## DNS-Auflösungsablauf

```text
DNS-Anfrage (example.uip)
    │
    ▼
Knoten: DomainId = BLAKE3-256("SCONE-DOMAIN-V1" || "example.uip")
    │
    ├── autoritativer DNS-Snapshot (RAM) ─► unbekannt/abgelaufen → NXDOMAIN
    │                                          (ohne den Kern zu berühren)
    ├── Shard-DNS-Cache (qname,qtype) ────► Treffer: geteilter Eintrag,
    │                                          träge Verifikation
    ├── lokaler Cache (SignedDnsRecord) ──► Verifikation (Signatur, Owner,
    │                                          sequence, record_hash ggü. Chain)
    ├── DHT get(DomainId) ────────────────► gleiche Verifikation
    │                                          (Fehlschlag → 30 s Negativcache)
    ▼
DNS-Antwort (genau dann, wenn gültig)
```

Der DNS-Snapshot (Existenz + Ablauf jeder lebenden Domain,
veröffentlicht nach jedem angewandten Block) garantiert, dass eine
Existenzentscheidung nie durch die Run-Schleife des Relays geht: der
Kern wird nur für tatsächliche Eintragsabrufe berührt.

## Entwickelbarkeit

- versionierte Präimages (`SCONE-DOMAIN-V1`, `SCONE-PUBKEY-V1`,
  `SCONE-OWNER-V1`, …): eine geänderte Ableitungsregel erzeugt einen
  neuen Namespace, statt den bestehenden zu brechen;
- `RecordData` erweiterbar über Varianten + `Unknown { type_code, data }`
  für zur Lesezeit unbekannte Typen;
- `PROTOCOL_VERSION` (`scone-protocol`) für Wire-Format-Brüche;
- die Chain speichert nur Hashes: große Daten bleiben im DHT.

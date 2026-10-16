# Methodik des Sicherheitsaudits — Scone (vor dem Mainnet)

Arbeitsdokument. Ziel: Vertrauen für den Produktiveinsatz.
Crate-übergreifende Referenzen: die harten Regeln der AGENTS.md
(keine Eigenbau-Kryptografie, striktes Dekodieren, neu berechnete
Hashes, Determinismus) sind die Anforderungsspezifikation des
Audits.

## 0. Umfang und Bedrohungen (bevor eine Zeile Code neu gelesen wird)

Gegnermodell: ein beliebig feindseliger p2p-Knoten (kann beliebige
Bytes über QUIC/UDP senden), ein feindseliger Tx-Einreicher, ein
neugieriger Relay-Betreiber, ein gestohlenes Datenverzeichnis. PER
KONSTRUKTION AUSGESCHLOSSEN: der Endclient (DNS ist von Natur aus
öffentlich), die globale Verfügbarkeit (Chaos bereits getestet).

Asset-Tabelle, nach Kritikalität geordnet:
1. **Eigentum** (Zustands-Chain: Register/Update/Transfer/Assign) —
   ein Falsch-Positiv = Domain-Diebstahl.
2. **Signatur/Schlüsselbesitz** (Keystore, Ed25519,
   OwnerId-Ableitung).
3. **Validierung unvertrauenswürdiger Eingaben** (Wire-Dekodierung,
   Grenzen, Panics).
4. **Ressourcen-DoS** (DNS-Cache, Mempool, DHT, Sync).
5. **Determinismus/Konsens** (Reorg, GC, Zustandsauferstehung).

## 1. Automatisiertes statisches Audit (Tag 1 — mechanisch)

- `cargo audit` (Abhängigkeits-CVEs: libp2p/quinn/rustls/redb/ed25519).
- `cargo geiger` + Durchsicht von `unsafe` (erwartet ≈ 0).
- `cargo clippy --workspace --all-targets -- -D warnings`
  (bereits grün, als CI-Schranke halten).
- Fuzzing der Dekodierungsgrenzen (Tag 2-3):
  `cargo-fuzz` auf `scone_protocol::decode_complete` (Message,
  Transaction, Block, SignedDnsRecord, Checkpoint) — das ist DIE
  Fläche, auf der „nie panisch bei unvertrauenswürdigen Daten“
  gelten muss. Budget: 24 h kontinuierliches Fuzzing, Korpus aus
  bestehenden Tests gesät. Exit-Kriterium: null Abstürze, null
  Timeouts, Abdeckung der Dekodierzweige > 95%.
- Fuzzing des DNS-Wire-Parsers (dns.rs `parse_query`,
  `extract_answers`) und des Eintragsdatei-Parsers (domain.rs).

## 2. Gezielte manuelle Durchsicht pro Komponente (Tag 4-8)

Nach Asset-Kritikalität geordnet, 1 Komponente = 1 Checkliste:

### 2.1 scone-crypto + Identitätsableitung (Tag 4)
- OwnerId = BLAKE3(pubkey): Byte-Kanonisierung prüfen (keine
  mehrdeutige Darstellung → 2 Schlüssel, 1 Owner).
- Ed25519: strikte DESERIALISIERUNG (nicht-kanonischer Schlüssel
  abgelehnt?), Verifikation mit Kontext-Trennung (jede Verwendung
  hat ihren Domänen-Tag: SCONE-TX-SIG-V1, SCONE-BLOCK-V1,
  DHT-Eintrag — kein protokollübergreifendes Replay möglich).
- Keystore: Passphrase → Schlüssel (KDF? Iterationen?),
  Nullung, Dateiberechtigungen, Padding-Oracle auf dem Geheimtext.

### 2.2 Transaktionsanwendung (Tag 5) — der Kern
Für Jede Transaction-Variante (heute 13):
- Ist die Tx unter Replay idempotent (eindeutige Txid,
  Nonce/Sequenz)?
- Geht die Eigentumsprüfung jedem Effekt voraus? (Signer == Owner
  on-chain gelernt, nie aus der Tx selbst geglaubt.)
- UpdateDomain-Sequenz: exakt current+1 (nicht ≥) — Ablehnung von
  Lücken und Ablehnung von Replays prüfen.
- Renew: Zeitgrenzen (≤ 3 Jahre, > now) — die Zeit kommt aus dem
  BLOCK, nie aus der lokalen Uhr (Regel bereits gesetzt, jede
  Aufrufstelle prüfen).
- AssignDomain/TransferTld: wird der Public-Key des Empfängers
  GELERNT (die Pubkey-lose-Pool-Falle — vgl. Skill) auf allen
  Pfaden.
- GC/Ablauf: keine Auferstehung möglich (das
  Precheck-OK/Lookup-OK/Push-scheitert-Paradox) —
  `gc_expired_journaled` prüfen.

### 2.3 Netzwerkschicht (Tag 6)
- Jedes entfernte Byte durchläuft einen strikten begrenzten
  Dekoder VOR der Allokation (MAX_* aus limits.rs) — Suche nach
  unbegrenztem `Vec::with_capacity(input_len)`.
- Requery/Sync GetBlocks: Grenzen pro Peer und pro Zyklus
  (bereits: 128 Blöcke/Anfrage) — Abwesenheit von
  Neu-Download-Schleifen prüfen.
- Gossip Record: Chain-Neuverifikation VOR Cache/Gossip (durch
  meine record_dht-Tests verifiziert) — auch die Fanout-Obergrenze
  prüfen.
- kad DHT: Kann ein feindseliger Peer die Speicherung von N
  Einträgen erzwingen (Quote pro Peer?) oder die gezielte
  Verdrängung ehrlicher Einträge?

### 2.4 DNS-Fläche (Tag 7)
- Cache: Injektion unmöglich (kanonischer (qname,qtype)-Schlüssel —
  keine Groß-/Kodierungs-Kollision), hartes Speicherbudget
  (getestet), kein Cache-Poisoning über den Fallback-Pfad (TXID +
  Fragen-Echo — verifizieren, dass der Upstream nicht aus der Ferne
  gespooft werden kann).
- Antworten: ehrliches AA, ehrliches TC, keine Verstärkung
  (begrenztes Antwort/Anfrage-Verhältnis — MAX_INFLIGHT_UDP bereits
  da, TCP-Replikation prüfen).

### 2.5 Speicherung & Genesis (Tag 8)
- redb: Festplattenkorruption → typisierter Fehler, keine Panik
  (der Diskfail-Test existiert auf der Signer-Seite — das
  Relay-Äquivalent?).
- Genesis: der Hash deckt das ganze Manifest ab; zwei verschiedene
  Geneses = Datenverzeichnis-Verweigerung (getestet); die
  Unmöglichkeit verifizieren, NACHTRÄGLICH eine Genesis in ein
  synchronisiertes Datenverzeichnis zu injizieren.

## 3. Gegnertests (Tag 9-12) — schreiben, was fehlt

Für jeden Befund der manuellen Durchsicht ein Test, der ihn beweist:
- feindseliger Knoten in der Simulation (SimNet mit einem Knoten,
  der malformede/nichtübereinstimmende Nachrichten sendet) — SimNet
  um eine Rolle „passiv byzantinisch“ erweitern;
- echtes 30-Knoten-Mesh mit 3 feindseligen Knoten (zufällige Bytes,
  Bursts ungültiger Txs, gefälschte DHT-Einträge) — die
  Chaos-Batterie existiert, die Angreiferrolle hinzufügen;
- Kreuzprüfungen: zwei Implementierungen der Dekodierung (das
  protocol.md-Dokument ist NORMATIV — ein Auditor implementiert das
  Wire NUR nach dem Dokument und vergleicht Bytes; jede Abweichung
  = Dokument- ODER Code-Bug).

## 4. Externes Audit (falls Budget) — nach 1-3

Minimal profitabler Umfang für einen Anbieter: scone-crypto,
scone-protocol (Wire), Tx-Anwendung (state.rs), ~8-10 k LOC.
Format: 2 Wochen, 1 Senior-Krypto-Auditor + 1 Reviewer. Lieferung:
Bericht mit Schweregrad, PoCs, geprüften Fixes. Auf die bestehende
normative Dokumentation (docs/technical) als Grundlage der
Anforderungsspezifikation stützen — das ist ein echter
Kostenvorteil.

## 5. Exit-Kriterien (Mainnet Go/No-Go)

- [ ] cargo-fuzz 24 h ohne Absturz auf den 5 Dekodern
- [ ] Durchsicht 2.2 vollständig: jede Tx-Variante hat ihr Blatt
      (Eigentum/Replay/Zeit/Effekte), vom Reviewer abgezeichnet
- [ ] 30-Knoten-Mesh inklusive feindseliger: 48 h ohne
      Spitz-Divergenz oder Cache-Korruption
- [ ] Dokument/Code-Abweichung: null (durch die duale
      Wire-Implementierung verifiziert)
- [ ] Blockierende CVEs: null (cargo audit)
- [ ] Externes Audit des Krypto-/Wire-/Tx-Umfangs erfolgt (falls
      budgetiert)

## Empfohlene Ausführungsreihenfolge

1→2.1→2.2 (der Kern, 60% des Risikos)→2.3→2.4→2.5→3 parallel zu
jeder Durchsicht (der Gegnertest wird am Tag des Befunds
geschrieben, nicht danach).

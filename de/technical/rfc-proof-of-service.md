# RFC — Betriebsnachweis und Finalitätskomitee (Proof-of-Service)

> Status: NORMATIV — Ende bis Ende implementiert (Schritte 1 bis 5:
> Core/Protokoll/Blockchain/Netzwerk/CLI; dieses Dokument ist die
> konsolidierte Form). Jede Regel unten ist gegen den Code
> verifiziert; die zitierten Konstanten sind die kodierten Werte.
> Datum: Oktober 2026 (normative Konsolidierung). Berührt:
> `scone-core`, `scone-protocol`, `scone-blockchain`,
> `scone-network`, `scone` (CLI).

## 1. Motivation

Der Finalitätspool ist der Owner-Pool (Eigentümer lebender
Domains/TLDs). Besitz gewährt einen Sitz — **Betrieb** gewährt
nichts. Dieser RFC bindet das Recht, Checkpoints zu signieren, an
den Betriebsnachweis eines Online-Relays, ohne je den Besitz zu
bedingen:

- **Bewahrte Freiheit**: eine private TLD bleibt ohne
  Aktivitätsbedingung zuweisbar, solange ihre Laufzeit verlängert
  wird (`RenewTld`, 3-Jahres-Laufzeit). Nur das *Checkpoint-Signieren*
  wird durch Präsenz verdient.
- **Anreiz zu betreiben**: eine private TLD (oder jeden Sitz)
  betreiben = im Komitee sitzen. Der Betreiber nimmt am Aufrecht-
  erhalten des Netzwerks teil, und das Protokoll erkennt es an.
- **Messbare Dezentralisierung**: das Komitee wird zur Menge der
  *tatsächlich online* betriebenen Knoten, nicht schlummernder
  Schlüssel.

## 2. Überblick

Eine leichtgewichtige `Heartbeat`-Transaktion (Diskriminant
`0x71`), signiert von einem **Betreiber**-Schlüssel, einmal pro
Epoche emittiert. Der Zustand hält ein gleitendes Fenster
jüngster Heartbeats. Das Komitee einer Epoche wird **aus der
Teilmenge des Pools gewählt, deren effektiver Betreiber einen
frischen Heartbeat hat**. Ohne Heartbeat: kein Signieren, aber kein
Verlust von Eigentumsrechten.

## 3. Die Heartbeat-Transaktion

```
Heartbeat {
    operator: PublicKey,     // Ed25519-Schlüssel des Betreibers (der Signer)
    owner: OwnerId,          // Identität NEU ABGELEITET vom Operator (nie geglaubt)
    tip_seen: [u8; 32],      // zuletzt gesehene Spitze des Knotens
    height_seen: u64,        // entsprechende Höhe
    timestamp: u64,          // Unix-Sekunden
} + Ed25519-Signatur über der kanonischen Nutzlast (Netzwerk injiziert)
```

Signierter Rumpf (Wire, `scone-protocol`):
`operator[32] ‖ owner[32] ‖ tip_seen[32] ‖ height_seen.v ‖
timestamp.v ‖ operator[32]` (der Schlüssel des Signers erscheint am
Ende des Rumpfes, wie bei jeder Tx) `‖ signature[64]`, also ~230
Bytes kodiert — **kein PoW**: die Kosten sind kontinuierliche
Präsenz (ein Heartbeat pro Epoche, über Monate).

Der Signer IST der Betreiber: `owner` wird immer aus `operator` neu
abgeleitet (nie aus der Eingabe übernommen), und das Dekodieren
lehnt mit einem typisierten Fehler (`InvalidOwner`) einen
Signaturschlüssel ab, der vom Kopf-Feld `operator` abweicht.

- `tip_seen`/`height_seen` dienen der Beobachtbarkeit (Erkennung
  fehlerhafter oder zurückliegender Betreiber); sie beeinflussen
  die Gültigkeit NICHT (ein leicht zurückliegender Knoten ist
  weiterhin ein lebender Knoten).

### 3.1 Reine Validierung (scone-core)

1. Gültige Ed25519-Signatur über der kanonischen Nutzlast des
   Netzwerks.
2. `owner` == aus `operator` abgeleitete Identität (sonst
   `InvalidOwner`) — beim Kodieren, beim Dekodieren UND von
   `validate()` geprüft.

### 3.2 Zustandsregeln (Anwendung, scone-blockchain)

Bei Anwendung eines Heartbeats in einem tragenden Block der Höhe
`block_height` (jede Zurückweisung geht jeder Mutation voraus):

1. **Cross-Epochen-Anti-Replay**: `timestamp` strikt steigend pro
   Betreiber — typisierte Zurückweisung `StaleHeartbeat`, wenn
   `timestamp <= operator_last_hb[operator]`.
2. **Einzigartigkeit pro Epoche**: nur ein Heartbeat akzeptiert pro
   (`operator`, Epoche des tragenden Blocks) — Duplikat mit
   typisiertem Fehler abgelehnt (`HeartbeatDuplicate`). Die Epoche
   ist `block_height / epoch_min_blocks` (dieselbe Definition wie
   Finalität, nie eine neue Konstante).
3. Journlierte Einfügung in das Fenster + Aktualisierung von
   `operator_last_hb` (bei Reorg rückgängig, bit-exaktes Undo).

### 3.3 Anti-Spam

- 1 akzeptierte Tx pro Betreiber pro Epoche — der maximale Fluss
  ist `pool_size / epoch_secs`, vernachlässigbar.
- Keine Gebühren (Prä-Ökonomie): die Rate ist strukturell begrenzt.

## 4. Betriebsdelegation

Die Verknüpfung Sitz → Betreiber läuft über zwei dedizierte
Transaktionen:

- `SetTldOperator { tld_id, operator }` (Diskriminant `0xA9`),
  signiert vom **aktuellen** Eigentümer der **privaten** TLD (eine
  offene TLD hat niemanden, der legitim signieren könnte —
  typisierte Zurückweisung `TldIsOpen`). `operator: None` =
  Widerruf, zurück zur Selbstbedienung durch den Schlüssel des
  Eigentümers; sofort widerrufbar.
- `SetDomainOperator { domain_id, operator }` (Diskriminant
  `0x5E`), signiert vom aktuellen Eigentümer der Domain. Dieselbe
  Semantik.
- `RegisterTld` / `RegisterDomain` / `AssignDomain` tragen KEIN
  Feld `operator`: die Verknüpfung Sitz → Betreiber ist immer ein
  rezenter On-Chain-Fakt, an der Spitze aufgelöst.

Die Betreiber-Option auf dem Wire ist kanonisch: `0x00` = `None`
(Selbstbetrieb), `0x01` = roher 32-Byte-Schlüssel; jeder andere
Wert wird beim Dekodieren abgelehnt.

Der Zustand hält:

```
seat_operators: HashMap<OwnerId, OperatorKey>   // pro Sitz
operator_seats: HashMap<OperatorKey, u64>       // begrenzter Zähler
operator_last_hb: HashMap<OperatorKey, u64>     // zuletzt akzeptierter ts
```

Anwendungsregeln (`apply_seat_operator`, journliert):

- **Idempotenz**: erneute Delegation an denselben Betreiber (oder
  Widerruf einer nicht existierenden Delegation) ist ein nicht
  journliertes No-Op (bit-exaktes Undo erfordert es).
- **Anti-Vermietungs-Schranke**: ein Betreiber kann nur von **einem
  einzigen Sitz** gleichzeitig delegiert werden
  (`MAX_SEATS_PER_OPERATOR = 1`). Ein zweiter Sitz, der an einen
  bereits vergebenen Betreiber delegiert → typisierte Zurückweisung
  `OperatorAlreadyDelegated`. Begründung: ohne Schranke verkauft
  ein großer Betreiber das Signieren in Massen und erschafft
  Zentralisierung neu — genau das, was das gewählte Komitee
  vermeiden soll.

## 5. Frische-Fenster

### 5.1 Gleitendes Fenster

```
recent_heartbeats: BTreeMap<(epoch, OperatorKey), ()>
```

- Eintrag bei Anwendung eines gültigen Heartbeats eingefügt (§3.2).
- Deterministische Räumung bei JEDER Blockanwendung
  (`gc_heartbeats_journaled`): jeder Eintrag der Epoche `e` mit
  `e + HEARTBEAT_WINDOW_EPOCHS <= epoch` wird entfernt (journliert —
  bei Reorg rückgängig, wie die Ablauf-Warteschlangen).
- `HEARTBEAT_WINDOW_EPOCHS = 2`: die aktuelle Epoche + die
  vorherige — ein Betreiber darf EINE Epoche verpassen, ohne die
  Frische zu verlieren, die zweite lässt sie ablaufen.
- **Strukturelle Speicherobergrenze**: das Fenster hält höchstens
  `HEARTBEAT_WINDOW_EPOCHS × operator_count` Einträge (einer pro
  Betreiber pro Epoche, Tupel-Schlüssel); `operator_last_hb` ist
  durch die Anzahl der Betreiber begrenzt und wird nie geräumt (die
  Anti-Replay-Regel ist monoton).

### 5.2 Frische

„Frisch“ = Präsenz eines Eintrags `(e, operator)` für `e` in
`[epoch − HEARTBEAT_WINDOW_EPOCHS + 1, epoch]`. Da der Zustand bei
jedem Block räumt, IST Präsenz in `recent_heartbeats` Frische
(`operator_is_fresh`).

### 5.3 Effektiver Betreiber

Der **effektive Betreiber** eines Sitzes ist `seat_operators[seat]`,
falls eine lebende Delegation existiert, andernfalls der Schlüssel
des Eigentümers (`effective_operator`). Der Sitz signiert unter
diesem effektiven Schlüssel — eine Delegation verschiebt das
Signierrecht, nie das Eigentum.

## 6. Komiteewahl

```
eligible = { effektive Betreiber der Sitze des eingefrorenen Pools
             (verbannter Schlüssel ausgeschlossen) mit frischem Heartbeat }

if |eligible| >= MIN_FINALITY_COMMITTEE_SIZE (= 4) :
    committee = select_committee(seed, eligible)   // bestehende Ziehung
else :
    committee = select_committee(seed, pool)       // sichere Degradation
```

Normative Details (verifiziert in `finality.rs`, spezifiziert in
`/docs/en/technical/blockchain.md` § „Sitze und das
Finalitätskomitee“):

- der `pool` ist der **beim letzten finalisierten Checkpoint
  eingefrorene** Pool (`FinalizedBase.pool` — nie der lebende
  Zustand): eine lokale Sicht würde zwischen Knoten divergieren,
  und das Quorum würde sich nie bilden;
- der `seed` leitet sich vom Inhalts-Hash des vorherigen
  Checkpoints ab (`signing_hash`);
- jeder Sitz des eingefrorenen Pools wird durch den Zustand an der
  Spitze zu seinem EFFEKTIVEN Betreiber aufgelöst (Delegation und
  Heartbeats sind rezente On-Chain-Fakten: zwei Knoten auf
  derselben Spitze berechnel dieselbe Menge); verbannte Schlüssel
  bleiben ausgeschlossen;
- die Ziehung läuft über die sortierte/deduplizierte Menge
  effektiver Schlüssel (`select_committee` unverändert).

**Explizite und sichere Degradation**: ein bootstrappendes
Netzwerk (kein Heartbeat) oder ein allgemeiner Ausfall der
Betreiber fällt auf den gesamten Pool zurück — die Finalität geht
konstruktiv nie verloren. Das ist die Regel „nie verklemmen“ des
Projekts, hier angewandt.

### 6.1 Kalter Bootstrap

Epoche 0 (und solange weniger als `MIN_FINALITY_COMMITTEE_SIZE`
Betreiber frisch sind): der gesamte Pool. Die Einschränkung
verschärft sich natürlich, während Betreiber emittieren. Keine
besondere Übergangsperiode zu konfigurieren.

## 7. Automatische Emission (scone-network)

In jedem Produktionszyklus (`Wake::Produce` — kein neuer Timer)
prüft das Relay, ob eine EPOCHEN-GRENZE überschritten wurde
(`height / epoch_min_blocks`, dieselbe Definition wie Finalität).
Im ersten Zyklus einer neuen Epoche emittiert es einen Heartbeat,
wenn:

1. ein Ankerschlüssel bewaffnet ist (`--anchor-key` oder auto),
   UND
2. dieser Schlüssel selbst einen lebenden Sitz hält (LOKALE Prüfung:
   der Schlüssel ist im On-Chain-Berechtigungspool — Eigentümer
   lebender Domains/TLDs, verbannte Schlüssel ausgeschlossen).

Lokales Anti-Spam: höchstens EINE Emission pro Epoche pro Knoten
(`last_emitted_epoch`-Memo, lokaler Speicher, nie Chain-Zustand);
das Memo wird erst nach erfolgreicher Mempool-Injektion genommen —
ein Fehlschlag versucht es im nächsten Zyklus erneut. Nichts hier
ist fatal: im schlimmsten Fall wartet der Heartbeat auf den
nächsten Zyklus.

Ein **delegierter Betreiber** (der nicht selbst Eigentümer ist) ist
von der automatischen Emission nicht abgedeckt: er emittiert über
den CLI-Builder von §8 (manueller oder Air-Gap-Betrieb). Der
Heartbeat wird NICHT notwendigerweise vom Emittenten selbst in
einen Block produziert: er ist eine Tx wie jede andere (Mempool,
Produktion durch jedermann).

## 8. CLI (`scone`)

Der Offline-Heartbeat-Builder (mit der `tx`-Fläche entfernt) wurde
mit der Offline-Tx-Fläche entfernt (cli-simplify-v1). Relays
emittieren ihren eigenen Heartbeat automatisch (§7); manuelle
Air-Gap-Emission, falls je wieder nötig, würde als eigener Befehl
zurückkehren. Standards waren: `tip_seen` = 32 Null-Bytes (nur
Beobachtbarkeit), `height_seen` = 0, `timestamp` = now.
- Wie bei jeder Tx **bindet** `tx sign` Operator und Owner an den
  ECHTEN Signaturschlüssel neu (Eingabewerte nie geglaubt); `tx
  verify` validiert lokal.

## 9. Beobachtbarkeit

- `NodeStatus` (Wire-p2p-Nachricht `0x0d`) trägt
  `fresh_operators: u32` (Varint): die Anzahl der Betreiber mit
  frischem Heartbeat — die lebende Teilmenge des für das Komitee
  berechtigten Pools (effektive Betreiber der Sitze des Pools,
  verbannte Schlüssel ausgeschlossen, sortiert/dedupliziert). Vom
  Relay aus `state().fresh_operators(height)` gefüllt; von `scone
  status` als `fresh operators: N` angezeigt.
- Kampagnenargument: „eine private TLD betreiben = im
  Finalitätskomitee des dezentralen DNS sitzen“ — on-chain
  verifizierbar.
- Künftige Fläche (unspezifiziert): `operator_status` — pro Sitz
  der delegierte Betreiber, Frische (frisch / abgelaufen / nie
  gesehen), letzter `tip_seen`. Das lokale Kontroll-RPC wurde
  entfernt (nur-p2p-Knoten); die Fläche ist noch zu definieren.

## 10. Was dieser RFC NICHT tut

- **Kein Verlust des Besitzes** bei Inaktivität (Option A — Zerfall
  der unbetriebenen TLD — ist ausdrücklich abgelehnt).
- **Keine Belohnung** (Option C wartet auf eine wirtschaftliche
  Schicht).
- Der Heartbeat beweist nicht die *Qualität* des Dienstes
  (produzierte Blöcke, bediente DHT-Einträge) — nur Präsenz. Das
  Zählen echter Beiträge ist eine mögliche künftige Erweiterung,
  leichter spielbar.

## 11. Geschützte Invarianten

- Bit-exakte Determiniertheit: Fenster und Wahl sind reine
  Funktionen des angewandten Zustands (journliert, bit-exaktes
  Undo).
- Nie panisch bei Netzwerkdaten: malformed Heartbeat oder
  Delegation → typisierte Zurückweisungen (`InvalidOwner`,
  `StaleHeartbeat`, `HeartbeatDuplicate`,
  `OperatorAlreadyDelegated`, `TldIsOpen`, `NotTldOwner`,
  `NotOwner`).
- Besitz ist bedingungslos; nur das Checkpoint-Signieren ist
  bedingt.

## 12. Konstanten und Parameter (kodierte Werte)

| Name | Wert | Ort |
|---|---|---|
| `HEARTBEAT_WINDOW_EPOCHS` | 2 | `scone-blockchain/src/state.rs` |
| `MAX_SEATS_PER_OPERATOR` | 1 | `scone-blockchain/src/state.rs` |
| `MIN_FINALITY_COMMITTEE_SIZE` | 4 | `scone-core/src/consensus_params.rs` |
| `Heartbeat`-Diskriminant | `0x71` | `scone-protocol/src/transaction.rs` |
| `SetTldOperator`-Diskriminant | `0xA9` | derselbe |
| `SetDomainOperator`-Diskriminant | `0x5E` | derselbe |

Diese Werte sind Teil des Konsenses: ihre Änderung bricht die
Chain-Wiederspielbarkeit (Fenster/Räumung) oder die Wahl
(Komitee).

## 13. Implementierungsplan — Endzustand

| Schritt | Crate | Inhalt | Zustand |
|---|---|---|---|
| 1 | core + protocol | `Heartbeat`, `SetTldOperator`, `SetDomainOperator`, Wire, Validierung | erledigt |
| 2 | blockchain | gleitendes Fenster (journliert), gefilterter Pool, Wahl, ~15 Tests | erledigt |
| 3 | network | automatische Emission, Tx-Relay | erledigt |
| 4 | CLI | `tx build heartbeat`, Anzeige `fresh operators` in `scone status` | erledigt |
| 5 | docs | dieses RFC in normative Form konsolidiert, fr/en-Spiegel | erledigt |

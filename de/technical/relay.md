# Netzwerk-Relay (M4) — Architektur, Abläufe und CLI-Steuerung

> Normative Spezifikation des Scone-Netzwerk-Relays
> (`scone-network`). Jede Abweichung zwischen Dokumentation und
> Code = Bug (die Quelle der Wahrheit beheben).

## Überblick

Das **Relay** ist Scones voller P2P-Knoten: es unterhält die
kanonische Blockchain (RAM + `NodeStore` redb), relait
Transaktionen und Blöcke, produziert Blöcke (Devnet-Modus), bedient
das Synchronisierungs-Protokoll, veröffentlicht und löst
`SignedDnsRecord`s in einem Kademlia-DHT auf und bietet das
**p2p-Protokoll** (`/scone/reqres/1`) an, das die CLI direkt abfragt
(RPC→p2p-Vereinheitlichung: der JSON-TCP-Kanal ist weg).

```text
                    ┌────────────────────────────────────────┐
                    │              scone relay               │
  P2P (libp2p)      │                                        │
  QUIC (udp,        │  Swarm libp2p                          │
  ephemeral port    │  ├─ identify  (Adress-Erkennung)       │
  by default) ◄─────┤  ├─ ping      (Lebendigkeit)           │
                    │  ├─ kad       (Eintrags-DHT)           │
                    │  └─ reqres    (scone-prot.-Nachrichten)│
                    │                                        │
                    │  Relay-Kern (einzige Task)             │
                    │  ├─ Blockchain (volle Validierung)     │
                    │  ├─ Mempool (begrenzt, Dedup nach TxId)│
                    │  ├─ BlockBuilder (Devnet-Produktion)   │
                    │  ├─ Fork-Parking (begrenzt, 2048)      │
                    │  └─ Sync-Treiber (gestaffelter Aufhol-) │
                    │     (lauf)                             │
                    │                                        │
  CLI ◄─────────────┤  reqres p2p (QUIC, dieselben Nachrichten)│
                    └────────────────┬───────────────────────┘
                                     │
                              NodeStore (redb)
```

`crates/scone-network` ist das **einzige async-Crate** im
Workspace: alles unterhalb (crypto → core → protocol → blockchain →
storage) bleibt synchron und rein.

## Verbindungs-Zulassung (P1.5)

Harte Grenzen auf Schwarm-Ebene (libp2p
`connection_limits`-Verhalten, der erste Eintrag von
`SconeBehaviour`):

- `MAX_TOTAL_CONNECTIONS = 128` — Obergrenze etablierter
  Verbindungen (eingehend + ausgehend) pro Relay; überschüssige
  eingehende werden ABGEWIESEN, nie in Warteschlange gestellt;
- `MAX_PER_PEER_CONNECTIONS = 4` — ein einzelner Peer kann das
  Gesamtkontingent nicht monopolisieren, indem er viele
  Verbindungen öffnet;
- `MAX_PENDING_INCOMING = 32` — Slowloris-Wächter: unauthentifizierte
  Halb-Verbindungen warten nie unbegrenzt.

Diese Grenzen ergänzen die bestehenden Grenzen pro Nachricht
(`MAX_MESSAGE_LEN` 1 MiB) und pro Budget (Sync `max ≤ 128` Blöcke,
DNS TCP 64 Verbindungen/512 B/5 s). Ein
Verbindungs-Flut-Angreifer erhält Zurückweisungen, kein
Speicherwachstum.

## Peer-Erkennung

Drei komplementäre Mechanismen, in dieser Prioritätsreihenfolge:

1. **Explizite Seeds** (`--bootstrap`, volle Multiaddrs) — haben
   immer Vorrang; das Bereitstellen einer nicht-leeren Liste
   ERSETZT alles andere. `--no-builtin-seeds` leert die
   eingebetteten Seeds für 100% private Deployments.
2. **Eingebettete Seeds** (`builtin_seeds(network)` in
   `scone-network/src/config.rs`) — bei jeder offiziellen
   Binary-Veröffentlichung eingefroren, pro Netzwerk. Ein
   unkonfiguriertes Relay joint dem Netzwerk allein. (Auf dem
   Testnet leer, solange das öffentliche Testnet nicht deployt
   ist.)
3. **Gelerntes Verzeichnis** (`<data-dir>/peers.json`) — jede über
   `identify` gelernte Adresse wird behalten (zuletzt gesehen; Cap
   von **2000 Einträgen** — Adressen, nicht Verbindungen, ~400 KiB
   RAM im Maximum — und eine **90-Tage-Altersgrenze**: ein vor
   mehr als 90 Tagen zuletzt gesehener Eintrag wird zur Schreibzeit
   verworfen und zur Ladezeit ignoriert, damit die 2000 Slots mit
   plausiblen Peers gefüllt bleiben statt mit
   Prä-Churn-Fossilien, deren einziger Beitrag eine
   Boot-Time-Out-Wahlverbindung wäre) und ~1×/Min am
   Produktions-Tick gespeichert. Beim Boot ohne Seed und ohne
   Bootstrap wird das Verzeichnis neu zusammengesetzt: das Relay
   findet seine üblichen Peers ohne Konfiguration. Es ist ein
   Cache: fehlende/korrumpierte Datei = leeres Verzeichnis, nie
   ein Fehler.
4. **LAN-mDNS** (`--mdns`) — Null-Konfigurations-Erkennung auf dem
   lokalen Link: benachbarte Relays auf demselben
   Netzwerksegment finden sich ohne jeden Seed (der
   Null-Konfigurations-Devnet-Pfad). **Standard: AN für Devnet,
   AUS sonstwo** (ein zurücksetzbares lokales Netzwerk ist per
   Design ein Erkennungs-Netzwerk; ein öffentliches Deployment
   multicastet nichts). Verhalten: Binden des Multicast-Sockets;
   ein Bind-Fehlschlag (kein Multicast auf der Schnittstelle,
   Sandbox-Umgebung) **deaktiviert** mDNS mit einer Warnung — nie
   fatal (Erkennung ist ein Komfort, nie eine Lebenslinie). Jeder
   `Discovered`-Nachbar speist die Kademlia-Tabelle
   (`kad.add_address`), exakt wie ein explizites Bootstrap minus
   der Konfiguration; `Expired`-Einträge werden von libp2p selbst
   verfolgt (die Verbindung schließt sich nach eigenem Timeout).
   Die Mitglieder `mdns`/`upnp` des Verhaltens sind `Toggle`s
   (bei Konstruktion aktiviert/deaktiviert,
   `build_behaviour(key, enable_mdns, enable_upnp)`).
5. **UPnP** (`--upnp`, Opt-in) — QUIC-Listen-Port-Freigabe durch
   den Heimrouter: der Knoten wird von außen erreichbar OHNE
   Circuit-Relay. Bei `NewExternalAddr` wird die externe Adresse
   als echte externe Adresse des Schwarms registriert
   (`add_external_address` — von identify/autonat gesehen, ein
   Transport-Relay-Kandidat). Kein IGD-Gateway (`GatewayNotFound`)
   oder nicht routbares Gateway (`NonRoutableGateway`): ein
   einfaches Info-Log, die Freigabe wird übersprungen — normale
   Fälle im Heim-LAN. Eine abgelaufene Freigabe (gescheiterte
   Erneuerung) ist eine Warnung.

mDNS folgt dem Netzwerk (ein explizites `--mdns`/`SCONE_MDNS`
gewinnt immer): **auf dem Devnet standardmäßig AN**, AUS auf
Testnet/Mainnet; UPnP bleibt **standardmäßig deaktiviert**
(`enable_mdns`/`enable_upnp` = `false` in `config.rs`, das Relay
schaltet mDNS für das Devnet ein): ein öffentliches Deployment
multicastet nichts und spricht mit keinem Router.

### Lokale Topologie-Optimierung (immer aktiv)

Das Relay optimiert seine Nachbarschaft aktiv über die reine
Engine `scone-topology` (siehe die Doku des Crates). **Immer
aktiviert**; die einzige exponierte Regelung ist ein **Profil**
(`--topology-profile`, Env `SCONE_TOPOLOGY_PROFILE`, Standard
`default`):

| Profil | min / Ziel / max | Auswertung | für wen |
|---|---|---|---|
| `tiny` | 2 / 4 / 6 | 120 s | Raspberry Pi, Heim-Box, 1-vCPU-VPS |
| `small` | 4 / 8 / 12 | 90 s | kleiner Server, Heim-NAS |
| `default` | 8 / 16 / 32 | 60 s | Standard-Server (Standard) |
| `high` | 16 / 32 / 64 | 45 s | souveräner Knoten, Backbone |
| `very-high` | 32 / 64 / 128 | 30 s | Founder-Hubs |
| `devtest` | 2 / 4 / 6 | 5 s | **NUR LABOR/TEST** (by Design instabil) |

```
scone relay --topology-profile tiny
```

Die Schutzparameter (Hysterese 0,15, Abklingzeit 600 s, Gnade
120 s, Messung 15 s) sind überall identisch: sie bewachen die
Stabilität und haben keinen Grund, mit der Knotengröße zu skalieren.
Ein unbekannter Name ist ein typisierter Start-Fehler.

Eine inkonsistente Konfiguration (`min > Ziel > max`, Schwelle
außerhalb `[0,1]` — `NaN` eingeschlossen —, Null-Intervall) ist ein
**typisierter Start-Fehler**, nie stilles Verhalten.

**Sicht (`TopologyView`)** — von Schwarm-Ereignissen gespeist:

- erfolgreicher `ping` → RTT-Beobachtung (Median-Fenster von 8
  Stichproben);
- gescheiterter `ping` → Zuverlässigkeits-Strafe;
- `identify` empfangen → Adressen (≤ 4/Peer) + angekündigte
  Protokolle (Bitmaske kad/reqres/identify/relay);
- Verbindung etabliert → verbundener Zustand, Richtung
  (eingehend/ausgehend), Alter, erkanntes Circuit-Relay;
- Verbindung geschlossen → Strafe + Churn-Zähler NUR wenn
  unaufgefordert (eine freiwillige Verdrängung zählt nicht);
- ausgehender Wahl-Fehlschlag → Zuverlässigkeits-Strafe.

Die Sicht ist begrenzt (500 Peers; darüber hinaus werden nur
verbundene Peers memoriert).

**`kad_required` aus den kbuckets** — vor jeder Auswertung
durchläuft das Relay die Kademlia-Routing-Tabelle: ein Peer ist
geschützt, wenn er `Connected` in einem Bucket ist oder
`Disconnected` in einem GESÄTTIGTEN Bucket (k = 20). Ein
geschützter Peer wird NIE zur Verdrängung gewählt.

**Optimierungs-Schleife** — im Auswertungs-Intervall des Profils
(60 s unter `default`; siehe Tabelle oben):

1. `kad_required` neu berechnen (kbuckets);
2. Engine-Auswertung (Hysterese: 120-s-Gnadenfrist,
   Mindest-Score-Verbesserung von +15%, Verdrängungs-Abklingzeit
   600 s);
3. Entscheidung angewandt: `AddCandidate` → den Kandidaten wählen;
   `Replace` → den verdrängten Peer schließen (EINMAL —
   progressiv, kein Looping), dann den Ersatz wählen;
4. ein **Metriken**-Schnappschuss wird geloggt (INFO-Stufe):
   Zähler bekannt/verbunden/eingehend/ausgehend/Circuit/geschützt,
   Mittel/Median-RTT, verschiedene IP-Buckets, maximale
   Konzentration, Churn (Ersetzungen/Trennungen).

Der Nachbarschafts-Score ist NIE eine Vertrauensgröße (siehe die
`scone-topology`-Doku): keine Konsens-Entscheidung leitet sich
davon ab.

### Erreichbarkeit (AutoNAT) und Transport-Relay (Circuit-Relay v2)

Jedes Relay betreibt AutoNAT (Client + Server): der Knoten bittet
seine Peers, eine Verbindung ZURÜCK zu ihm zu versuchen; das
Urteil (`reachable`: ja = von außen erreichbar, nein = hinter NAT,
unbekannt = noch nicht getestet) wird alle 5 Min aufgefrischt und
in `NodeStatus` exponiert (Feld `nat_reachable`, Wire-Varint
0/1/2) — die CLI zeigt es (`scone status`). Private Adressen zählen
(`only_global_ips = false`): ein LAN-Mesh testet seine
LAN-Erreichbarkeit.

Ein erreichbares Relay aktiviert Circuit-Relay v2 **Hop** (es kann
Verkehr für NATed Peers relaien) und **DCUtR**
(Lochbohrung: zwei über Circuit verbundene Knoten versuchen eine
direkte Verbindung und geben das Relay frei). Ein als NATed
eingeschätztes Relay fordert automatisch eine Reservierung beim
ersten bekannten erreichbaren Peer an und hört durch es (Adresse
`/p2p/<relay>/p2p-circuit`).

Produktionsregel: ein gerelayter NATed Knoten sollte nicht
Block-Produzent werden (beeinträchtigte Latenz/Verfügbarkeit) —
die PoS-Regel (Sitz = lebende Domain) verbietet es noch nicht; als
betriebliche Anforderung vor dem öffentlichen Testnet zu
dokumentieren.

## Transport und libp2p-Protokolle

| Protokoll | Name | Rolle |
|---|---|---|
| QUIC (quic-v1) | — | Haupttransport (udp) |
| identify | `/scone/id/1` | Peer-Adress-Erkennung (füllt die Kademlia-Tabelle) |
| ping | libp2p-Standard | Lebendigkeit |
| Kademlia | `/scone/kad/1` | DHT: Schlüssel = `DomainId` (32 B), Wert = kanonische Kodierung des `SignedDnsRecord` |
| Request-Response | `/scone/reqres/1` | scone-protocol-Anwendungsnachrichten |

### Anwendungs-Frame (`/scone/reqres/1`)

Jedes Frame ist `len: u32 BE || bytes`, wobei die Bytes die
kanonische Kodierung einer `Message` aus `scone-protocol` sind (der
bestehende Codec — kein zweites Format). Die Länge wird durch
`limits::MAX_MESSAGE_LEN` begrenzt **vor** jedem Lesen oder jeder
Allokation: ein Peer, das ein größeres Frame ankündigt, wird ohne
Speicherkosten abgelehnt.

Ausgetauschte Nachrichten (eine pro Request/Response-Roundtrip):

| Nachricht | Richtung | Antwort |
|---|---|---|
| `Hello { version }` | Handshake | `Hello` (lokale Version) |
| `Ping` / `Pong` | Lebendigkeit | `Pong` |
| `GetBlock { hash }` | Abfrage | `Block` (RAM-Fenster) oder `Pong`, falls unbekannt |
| `GetBlocks { start, max }` | Sync | **ein** `Block` (Höhe `start`), oder `Pong`, falls keiner |
| `Block (b)` | Block-Relay | `Pong` (Ack) |
| `Transaction (tx)` | Tx-Relay | `Pong` (Ack) |
| `GetRecord { domain_id }` | lokaler DHT | `Record` (lokaler Cache) oder `Pong` |
| `Record (r)` | Veröffentlichung | `Pong`, falls akzeptiert, sonst Fehler |

Regel: **genau eine Antwort pro Anfrage**;
Bestätigungsnachrichten sind `Pong`. Sync-Antworten tragen
jeweils einen Block — der Aufhol-Lauf iteriert auf der
Anfrager-Seite (siehe „Sync“). Das ist bewusst begrenzt: nie mehr
als ein Block in der Luft pro Peer.

## Relay-Verhalten

### Start

0. Netzwerk-Auflösung (`--network testnet|mainnet|devnet`,
   Testnet-Standard, solange das Projekt in Entwicklung ist —
   M8b). Das Datenverzeichnis ist **pro Netzwerk**
   (`~/.scone/<network>/`): ein nicht-leerer Store, der eine Chain
   eines anderen Netzwerks hält, wird abgelehnt (`Corrupted`);
1. Öffnen/Erstellen des redb-Stores (`<data-dir>/chain.redb`);
2. `load_chain(store, network)`: O(1)-Spitze + paginierte
   Domain-Zustände (RAM-Fenster: Genesis + Spitze);
3. Schwarm-Konstruktion (flüchtige Ed25519-Identität pro Sitzung —
   das Persistieren der Netzwerk-Identität ist eine offene
   Entscheidung);
4. Boot-Liste zusammenstellen (siehe
   [Peer-Erkennung](#peer-erkennung)): explizites `--bootstrap`
   (muss auf `/p2p/<peer-id>` enden) SONST die eingebetteten Seeds
   des Netzwerks SONST das gelernte Verzeichnis (`peers.json`);
5. jede Multiaddr wählen und in der Kademlia-Tabelle registrieren
   (eine tote Verbindung wird ignoriert — die anderen machen das
   Netzwerk aus).

### Empfang einer Transaktion (P2P oder CLI-`submit tx`-Einreichung)

1. striktes begrenztes Dekodieren (`MAX_MESSAGE_LEN`);
2. vollständige kryptografische Validierung
   (`validate_transaction`: Owner/Schlüssel-Bindung,
   `verify_strict` über der kanonischen Nutzlast);
3. **Anti-Replay ZUERST** (billig, zustandsunabhängig,
   definitiv), dann Zustands-Vorprüfung (billiger Spiegel der
   `push_block`-Regeln — PoW, das teuer ist, wird erst zur
   Push-Zeit verifiziert): ein wiederholtes `RenewDomain`
   antwortet `TxReplay`, nicht mit einem lügenden PoW-Fehler (sein
   Proof bindet das Ablaufdatum, an dem die erste Anwendung vorbei
   ist). Die Netzwerktrennung wird bei Schritt 2 durchgesetzt: Die
   Signaturnutzlast injiziert das Netzwerk des RELAYS, eine
   woanders signierte Tx scheitert mit `InvalidSignature` (Wire
   2026-09: kein `network`-Feld mehr);
   `RegisterDomain` → TLD des registrierten Namens (`UnknownTld`
   sonst, D1/M7c) **und offen** (`TldClosed` sonst, M8b), dann Name
   frei; `UpdateDomain` → bekannte Domain, exakte Sequenz
   `current + 1`, Owner; `RegisterTld` → freie TLD;
   `TransferTld`/`RevokeTld`/`SetTldOpen` → bekannte TLD + Owner-
   Signer; `AssignDomain` → TLD-Eigentümer + freier Name;
   `RenewTld` → bekannte private TLD + Owner, feste Laufzeit neu
   abgeleitet, Karenzfenster-Vorprüfung; `RenewDomain` → bekannte
   Domain + Owner, Verlängerungs-PoW-Vorprüfung (gebunden an das
   aktuelle Ablaufdatum) zur nächsten Höhe;
4. Mempool-Einfügung (feste Kapazität, Dedup nach `TxId` — das
   erneute Einreichen einer bereits im Pool befindlichen Tx ist
   ein No-Op);
5. Broadcast an Peers (außer dem Absender).

### Empfang eines Blocks (P2P, Sync oder selbst produziert)

1. `push_block`: bestehende vollständige Validierung (Parent =
   kanonische Spitze, Höhe, neu berechnete Merkle-Wurzel, jede
   Transaktion validiert dann angewandt — atomar pro Block);
2. falls akzeptiert: das Relay liest das Anwendungsergebnis
   (`push_block_with_gc` → Hash + durch die deterministische GC
   entfernte Domains) und persistiert über
   `store_block_with_removals` (atomares Delta-Anhängen: Block +
   Spitze + modifizierte Zustände — Upserts UND Entfernungen:
   abgelaufene Domains, widerrufene TLDs). Ein Neustart kann
   daher nie eine abgelaufene Registrierung auferstehen lassen;
3. Mempool-Abgleich (eingeschlossene Txs werden entfernt);
4. Broadcast an Peers (außer dem Absender).

Ein ungültiger Block wird mit einem typisierten Fehler abgelehnt;
das Relay läuft weiter. Keine Panik bei Netzwerkdaten. Schwere
Anwendungen (Argon2 + redb-Commit) laufen über
`tokio::task::block_in_place` (Sync-Treiber, p2p-Akzeptanz,
Produktion): die Laufzeit verschiebt bereite Tasks auf andere
Worker — das Anwenden eines Blocks blockiert nicht länger den
DNS-Dienst oder die Metriken.

### DNS-Snapshot (Veröffentlichung)

Nach JEDEM angewandten Block (p2p-Akzeptanz — Reorgs
eingeschlossen —, Sync-Treiber, Produktion) UND beim Boot (ein
Neustart darf lebende Namen nicht bis zum nächsten Block mit
NXDOMAIN beantworten), veröffentlicht das Relay den
**autoritativen DNS-Snapshot** auf dem `watch`-Feed, den die
DNS-Fläche liest (vollständige Spezifikation in `dns.md`):

- Spitzen-Verlängerung (der überwiegende Pfad): **O(berührt)
  Delta**, Copy-on-Write — die Transaktionen des Blocks nennen die
  berührten Domains, plus die Entfernungen der deterministischen
  GC (`AppliedBlock.gc_removed_domains`, zur Push-Zeit ausgewertet);
- Reorg / Boot: **voller Neuaufbau** (ein paginierter
  `ChainState::dns_facts`-Durchlauf).

### Block-Produktion (M5: signierter Produzent)

Alle `produce_interval` Sekunden (standardmäßig 2 s): wenn der
Mempool nicht leer ist, wird ein Block durch `BlockBuilder`
zusammengesetzt (`timestamp = now`), **von einem autorisierten
Produzenten signiert** (Nutzlast `SCONE-BLOCK-V1` in
`header.consensus`, siehe `/docs/en/technical/blockchain.md`) und
durchläuft exakt den Akzeptanz-Pfad eines empfangenen Blocks. Ein
Relay ohne Txs produziert nichts.

Produktionsschlüssel: der Ankerschlüssel (`--anchor-key`), wenn das
Relay damit bewaffnet ist — ein Anker ist ein autorisierter
Produzent —, andernfalls der feste Devnet-Schlüssel, gültig, solange
der Produzenten-Pool leer ist (offener Bootstrap: Genesis). Sobald
eine lebende Domain/TLD existiert, wird ein vom Devnet-Schlüssel
signierter Block abgelehnt (`InvalidProducer`).

### Anker-Schleife (Finalität, M5)

Mit `--anchor-key` (`.sconekey`-Schlüsseldatei, Passphrase über
`--anchor-passphrase-env`, Standard `SCONE_ANCHOR_PASS` — der
Bibliotheks-Standard `SCONE_ANCHOR_PASSPHRASE` von `scone-network`
wird über die Binärdatei nie erreicht, clap überschreibt ihn
immer):

- nach jedem akzeptierten/produzierten Block, wenn dieser Knoten
  Mitglied des aktuellen Komitees ist **und** das Komitee
  `MIN_FINALITY_COMMITTEE_SIZE` erreicht, signiert das Relay den
  vorgeschlagenen Checkpoint über seinen `SignerGuard`
  (`signer-state.dat` im Datenverzeichnis — ein Schlüssel = eine
  Stimme/Epoche, absturzsicher fail-closed) und gossipt es;
- empfangene Signaturen (`Message::Checkpoint`) werden nach
  `CheckpointData`-Hash akkumuliert (auf 256 Einträge begrenzt);
  bei `quorum_for(Komiteegröße)`-Mitglieder-Signaturen auf
  DENSELBEN Daten → `accept_checkpoint` + Broadcast des
  finalisierten Checkpoints;
- ein Relay ohne Ankerschlüssel bleibt **passiv**: es akkumuliert,
  finalisiert und relait, signiert aber nie;
- `GetCheckpoints` → das Fenster `checkpoint_window()` wird bedient;
- jeder Schleifen-Fehler wird geloggt und verschluckt — nie fatal;
- der Status (`GetStatus`) legt `checkpoints` und `finalized_epoch`
  offen.

### Fork-Parking und Reorgs (Sticky-Minority-Fix)

Ein Schwester-Block auf derselben Höhe oder ein Block, dessen
Parent nicht die aktuelle Spitze ist (`ParentNotTip`/
`UnknownParent`), geht NIE verloren: er wird **geparkt** (begrenzte
Map, `PARKED_CAP = 2048` Blöcke, ~KiB je Stück — begrenzter
Worst-Case-RAM; ein volles Parking ignoriert Neuankömmlinge
still). Hashes, die `accept_block` bereits angeboten wurden, werden
dedupliziert (`seen_blocks`): ein neu gegossipter verlierender
Block wird nicht neu ausgewertet.

Bei JEDEM `Produce`-Tick `reconsider_parked`:

1. lineares Ablassen geparkter Blöcke, deren Parent die Spitze
   wurde;
2. falls ein geparkter Zweig stärker ist als die aktuelle Spitze
   (Fork-Choice-Regeln UNVERÄNDERT: Finalitätsuntergrenze, längste
   Chain, niedrigster Hash), Adoption — das verdrängte Suffix wird
   selbst geparkt (BEIDE Seiten des Forks bleiben auswertbar).

Das befreit einen Knoten, der die verlierende Seite eines
gleichhohen Forks adoptiert hat: der gewinnende Block, einmal
angeboten dann abgelehnt, wird periodisch neu ausgewertet, statt
für immer vergessen zu werden (Chaos-30-Befund: zwei
Schwester-Blöcke 15 ms auseinander).

**Reorg-Persistenz**: das Adoptieren eines anderen Zweigs kann
nicht über das monotone Anhängen laufen
(`NonMonotonicHeight`). Das Relay wendet `rewind_to_height(Fork-
Punkt)` an, dann `persist_state_snapshot` (Neuschreiben des
lebenden Zustands aus dem Export der Chain), dann ein monotones
erneutes Anhängen des gewinnenden Suffixes (siehe `storage.md`).
Ein Persistenz-Fehlschlag wird geloggt und degradiert (heilt beim
Boot über Replay) — nie fatal.

## Sync (Aufholen)

Bei jeder Verbindung zu einem Peer fordert das Relay
`GetBlocks(start = lokale_Höhe + 1)` an. Jeder empfangene `Block`,
der die erwartete Höhe exakt vervollständigt, löst die Anfrage der
nächsten Höhe aus, bis zur Erschöpfung (`Pong` = nichts mehr) oder
`MAX_SYNC_ROUNDS` (10.000) Stapel — eine absolute Grenze selbst
gegenüber einem lügenden Peer. Empfangene Blöcke durchlaufen die
vollständige Validierung: ein bösartiges Peer kann nichts
Ungültiges akzeptiert bekommen.

### Eintrags-DHT

- **put** (API-/Web-Flow — der CLI-Hex-Kanal wurde entfernt):
  striktes begrenztes Dekodieren (`MAX_DHT_CACHE_ENTRY`),
  Verifikation gegen den On-Chain-Zustand (Owner, Sequenz, neu
  berechneter `record_hash` — siehe „Eintrags-Verifikation“), dann
  `kad.put_record` (lokale Speicherung + Replikation an die
  nächsten Peers, `Quorum::Majority` des Replikationsfaktors) und
  persistenter redb-Cache.
- **get** (`domain resolve`): `kad.get_record`; der Wartende ist an
  seine `QueryId` und die angeforderte `DomainId` gebunden — eine
  Antwort löst nur die passende Abfrage auf, und der
  Eintragsschlüssel muss exakt die angeforderte Domain sein
  (geprüft **vor** der On-Chain-Verifikation). Das Ergebnis wird
  dann gegen die Chain verifiziert, bevor es zurückgegeben wird.
  Obergrenze gleichzeitiger Anfragen: `MAX_DHT_WAITERS` (256);
  darüber sofortiger Fehler. Eine erfolgreich verifizierte
  Auflösung wird in den persistenten Cache geschrieben (M5):
  lokale Lesevorgänge (`domain_info`) bedienen sie dann ohne neue
  DHT-Abfrage.
- Kademlia läuft im Server-Modus, ohne Eintrags-TTL (die Chain
  regiert die Gültigkeit, nicht die Zeit).

### Eintrags-Verifikation (Regel „die Chain ist die Autorität“)

Ein aufgelöster oder empfangener `SignedDnsRecord` wird nur
akzeptiert/zurückgegeben, wenn:

1. die Domain on-chain registriert ist;
2. `record.owner` = aktueller On-Chain-Owner;
3. `record.sequence` = aktuelle On-Chain-Sequenz;
4. `record_hash(record.record)` (neu berechnetes BLAKE3) = der
   on-chain committete `record_hash`.

Sonst: stille Zurückweisung auf P2P-Seite, expliziter Fehler auf
CLI-Seite. Die Ed25519-Signatur des Eintrags wird beim Empfang
**vom Relay verifiziert** (`ResolveLocal`: der öffentliche
Schlüssel des Owners ist on-chain bekannt, die Signatur muss
aufschließen — ein von einem anderen Schlüssel signierter Eintrag
wird abgelehnt, `record rejected`); sie wird auch für künftige
Prüfer transportiert — das On-Chain-Commitment bleibt die
primäre Integritätskontrolle.

## Prometheus-Metriken

Ein HTTP-Endpunkt (Prometheus-Textformat 0.0.4, handgeschriebene
Exposition — keine neue Abhängigkeit), bedient von einer dedizierten
Task:

- **Standardmäßig AUS**: kein HTTP-Server ohne `--metrics-addr` (Metrics sind opt-in).
- `--metrics-addr ADDR:PORT`: bindet den Endpoint an diese Adresse (z. B. `127.0.0.1:9101` lokal, `0.0.0.0:9101` zum Exponieren).
- `--metrics-token TOKEN`: schaltet in den **privaten** Modus —
  `Authorization: Bearer *** required (Vergleich in konstanter
  Zeit, sonst `401`; der Token erscheint nie in Logs noch in der
  Exposition).
- Ein Bind-Fehlschlag **deaktiviert den Endpunkt** (nicht-fatale
  Warnung) — nie den Knoten.

Der effektive Bind wird auf stdout angekündigt
(maschinenlesbarer Vertrag, wie `p2p:`): eine Zeile `metrics:
<addr>`, sobald er hört.

**19 Zähler** (`counter`, monoton — Hot Paths:
`fetch_add(Relaxed)` auf einem geteilten `Arc<Metrics>`, nie eine
Konsens-Eingabe):

| Gruppe | Serien |
|---|---|
| Chain | `scone_blocks_applied_total`, `scone_blocks_rejected_total`, `scone_checkpoints_signed_total`, `scone_checkpoint_sigs_received_total` |
| Mempool | `scone_txs_admitted_total`, `scone_txs_rejected_total`, `scone_txs_evicted_grace_total`, `scone_txs_included_total` |
| DNS | `scone_dns_queries_total`, `scone_dns_cache_hits_total`, `scone_dns_cache_misses_total`, `scone_dns_local_chain_total`, `scone_dns_dht_answers_total`, `scone_dns_rejected_unverified_total` |
| DHT | `scone_dht_pulls_issued_total`, `scone_dht_negative_hits_total`, `scone_dht_negative_expired_total`, `scone_dht_negative_capped_total`, `scone_dht_cache_evictions_total` |
| Peer-Netzwerk | `scone_peer_connections_established_total`, `scone_peer_connections_closed_total`, `scone_peer_connections_incoming_total`, `scone_peer_dial_failures_total`, `scone_peer_blocks_received_total`, `scone_peer_transactions_received_total` |

**6 Gauges** (`scone_height`, `scone_mempool`, `scone_peers`,
`scone_domains`, `scone_finalized_epoch`, `scone_addr_book_entries`)
— die ersten fünf alle 15 s über den `RelayCommand`-Kanal
aufgefrischt (derselbe Pfad wie der CLI-`status` — einzige Quelle
der Wahrheit); `scone_addr_book_entries` wird direkt von der
Registrierung gepflegt (bei jeder Adressbuch-Einfügung aktualisiert,
Quelle der Wahrheit).

E2E-Test: `metrics_endpoint_public_and_private` (200 +
Exposition, 401 ohne/mit falschem Token, 200 mit Bearer).

## CLI-Steuerung (RPC→p2p-Vereinheitlichung)

Der JSON-TCP-RPC-Server wurde **entfernt**: die CLI (und die
axum-Brücke des Registrars) sprechen das p2p-Protokoll **direkt** —
dasselbe Wire (`/scone/reqres/1`), dieselben Handler auf
Relay-Seite, null paralleler Kanal. Jeder Befehl öffnet eine
QUIC-Verbindung zur Multiaddr des Knotens (`--peer`), tauscht EINE
`Message` aus, schließt.

Vom Relay bediente Steuer-Nachrichten:

- `GetStatus` → `NodeStatus { network, nat_reachable, tip,
  height, domain_count, mempool, peers, finalized_epoch,
  checkpoints, fresh_operators }` (angezeigt von `scone status` —
  `fresh_operators` = Betreiber mit frischem Heartbeat, RFC
  Proof-of-Service);
- `LookupName { name }` → `NameState` (On-Chain-Zustand, wie
  `lookup`);
- `GetOwnedNames { owner }` → `OwnedNames` (lebende Domains + TLDs
  eines `OwnerId`);
- `Transaction (tx)` → Einreichung mit freiem Gossip (das Relay
  sendet an Peers erneut aus);
- `GetRecord { domain_id }` / `Record (r)` — DHT-Auflösung und
  Veröffentlichung (wie `scone domain resolve` auf CLI-Seite);
- Anwendungs-Zurückweisung → `Message::Error(msg)` (erreicht den
  Client als lesbare Nachricht, nicht als EOF).

Der Client bringt einen minimalen libp2p-Schwarm hoch
(Request-Response + identify), 30-s-Anruf-Timeout (20-s-Roundtrip).
Standard-`--peer`:
`/ip4/127.0.0.1/udp/4001/quic-v1/p2p/12D3KooW…` (lokaler
Dev-Knoten; die volle Multiaddr mit der echten Peer-Id wird vom
Relay über stdout über die `p2p:`-Zeile angekündigt).

Der interne `RelayCommand`-Kanal überlebt für die eingebettete
DNS-Fläche (`ResolveLocal`, ein In-Process-Roundtrip, nie ein
DHT-Wartender — siehe `dns.md`).

## CLI

```
scone relay [--network testnet|mainnet|devnet] [--data-dir DIR] [--listen MULTIADDR] [--bootstrap MULTIADDR]... [--no-builtin-seeds]
            [--mdns] [--upnp] [--rpc ADDR (obsolet, ignoriert)]
            [--anchor-key PATH] [--no-anchor-auto] [--anchor-passphrase-env VAR] [--genesis FILE]
            [--geo-config FILE] [--geo-ip-table PATH] [--geo-country U16] [--geo-continent U16] [--geo-asn U32] [--geo-region U32]
            [--health-check] [--no-health-check] [--health-ttl-secs U64] [--health-port U16]
            [--topology-profile tiny|small|default|high|very-high|devtest]
            [--dns ADDR] [--dns-upstream ADDR]... [--dns-cache-limit U] [--dns-cache-max-mem SIZE] [--dns-max-inflight U]
            [--metrics-addr ADDR:PORT] [--metrics-token TOKEN]
scone status [--peer MULTIADDR]
scone domains [--identity ID | --pubkey HEX] [--peer MULTIADDR]
scone domain register NAME [--identity ID] [--dir DIR] [--passphrase-env VAR] [--peer MULTIADDR]
scone domain set NAME TYPE VALUE [TTL] [--identity ID] [--peer MULTIADDR]   # express single record
scone domain update NAME --file FILE [--identity ID] [--peer MULTIADDR]
scone domain get NAME [--peer MULTIADDR]
scone domain renew NAME [--identity ID] [--peer MULTIADDR]
scone domain resolve NAME [--peer MULTIADDR]

# --peer zeigt standardmäßig auf das lokale Relay (SCONE_PEER-Env überschreibt);
# --identity ist standardmäßig die einzelne Keystore-Identität, wenn genau
# eine existiert. Entfernt (Stubs schlagen Ersatz vor): show,
# submit, lookup, owned.
```

Jeder Steuer-Befehl spricht direkt QUIC mit der Multiaddr des
Relays (`--peer`, Standard lokaler Knoten — RPC→p2p-
Vereinheitlichung, kein JSON-TCP-Kanal mehr). Relay abwesend →
klare Meldung „p2p node unreachable — is a relay running?“,
Exit-Code ungleich null. Die Flagge `--rpc` bleibt, ist aber
**obsolet und ignoriert** (der RPC-Server wurde entfernt; nur
Skript-Kompatibilität).

### Umgebungskonfiguration (`.env`)

Beim Start lädt `scone` (über dotenvy, vor dem clap-Parsen) eine
`.env`-Datei aus dem aktuellen Verzeichnis: fehlende Datei = stilles
No-Op, nie ein Fehler. Die geladenen Werte werden die
**Umgebungsbasislinie** der Optionen, die eine Variable deklarieren
(`env = "…"` in `cli.rs`); **eine explizite CLI-Flagge gewinnt
immer** über `.env`, und `.env` gewinnt über die eingebauten
Standards:

```text
explizite CLI-Flagge  >  .env-Variable  >  eingebauter Standardwert
```

Die Datei `.env.example` an der Repository-Wurzel dokumentiert jede
Variable (nach `.env` kopieren und anpassen; `.env` ist
**gitignored** — nie eine echte `.env` committen). Variablen:

| Variable | Spiegel-Flagge | Semantik |
|---|---|---|
| `SCONE_NETWORK` | `--network` | `testnet` (Standard), `mainnet`, `devnet` |
| `SCONE_DATA_DIR` | `--data-dir` | Datenverzeichnis (Standard `~/.scone/<network>/`) |
| `SCONE_LISTEN` | `--listen` | P2P-Listen-Multiaddr (Standard: alle Schnittstellen, flüchtiger Port) |
| `SCONE_BOOTSTRAP` | `--bootstrap` | Bootstrap-Seeds, **wiederholbar** (eine Zeile pro Wert — dotenvy akkumuliert); ersetzt die eingebauten Seeds |
| `SCONE_GENESIS` | `--genesis` | Genesis-Datei (Founder-Knoten) |
| `SCONE_MDNS` | `--mdns` | LAN-mDNS-Erkennung (`true` = aktiviert, `false` = aus; Standard: AN auf Devnet, AUS sonstwo) |
| `SCONE_UPNP` | `--upnp` | UPnP-Port-Freigabe (`true` = aktiviert) |
| `SCONE_DNS_ADDR` | `--dns` | UDP-DNS-Server-Bind-Adresse |
| `SCONE_DNS_UPSTREAM` | `--dns-upstream` | rekursive Fallbacks, **wiederholbar** |
| `SCONE_DNS_CACHE_LIMIT` | `--dns-cache-limit` | Obergrenze der DNS-Cache-Einträge |
| `SCONE_DNS_CACHE_MAX_MEM` | `--dns-cache-max-mem` | DNS-Cache-Speicherobergrenze (z. B. `64MB`) |
| `SCONE_DNS_MAX_INFLIGHT` | `--dns-max-inflight` | Obergrenze gleichzeitiger DNS-Abfragen |
| `SCONE_METRICS_ADDR` | `--metrics-addr` | metrics endpoint address (`ADDR:PORT`)|
| `SCONE_METRICS_TOKEN` | `--metrics-token` | Bearer-Token des Metrik-Endpunkts |
| `SCONE_TOPOLOGY_PROFILE` | `--topology-profile` | Topologie-Profil (`tiny`/`small`/`default`/`high`/`very-high`/`devtest`; Standard `default`) |
| `SCONE_ANCHOR_KEY` | `--anchor-key` | Ankerschlüsseldatei (Checkpoint + Produktion) |

`SCONE_ANCHOR_PASS` ist keine Option einer Variable: Sie ist der
STANDARDNAME der **Variable, die die Anker-Passphrase hält**
(`--anchor-passphrase-env`, Standard `SCONE_ANCHOR_PASS` auf
CLI-Ebene) — das Relay liest die Passphrase aus der Variable, die
diese Flagge benennt.

**Anker-Auto (standardmäßig AN)**: ohne `--anchor-key` wird eine
Ankerschlüsseldatei automatisch im Datenverzeichnis erzeugt/
persistiert (`anchor.sconekey`, Passphrase `SCONE_ANCHOR_PASS`,
falls definiert, andernfalls zufällig erzeugt und nach `anchor.pass`
geschrieben) — das Relay wird Produzent, sobald sein Schlüssel
lebende Domains/TLDs besitzt (PoS-Berechtigung ist nie gratis).
`--no-anchor-auto` hält das Relay strikt passiv. Die Standard-Env-
Variable von `--anchor-passphrase-env` ist `SCONE_ANCHOR_PASS`.

`--network`: der Standard folgt dem BUILD (Mainnet auf
Mainnet-fähigen Binaries, sonst Testnet); `devnet` ist das
zurücksetzbare lokale Netzwerk (eigene
`scone-devnet`-Genesis, Fast-null-0..=2-Bit-PoW — das Löschen von
`~/.scone/devnet/` und ein Neustart setzt die Chain zurück). Setzt
die Genesis, die PoW-Schwierigkeiten und das
Standard-Datenverzeichnis (`~/.scone/<network>/`); eine Tx eines
anderen Netzwerks wird typisiert abgelehnt (`WrongNetwork`).

Die Offline-Hex-Fläche (die Unterbefehle `tx` und `submit`) ist weg
(remove-tx-surface-v2). Die CLI deckt die Nutzerpfade (`domain
register`, `domain renew`, `domains`) UND die M8b-Familie als
Erstklassige-Befehle: `tld open`, `tld transfer`, `tld revoke`,
`domain assign`, `domain transfer`. `register`-Claims mint ihr PoW
auf Netzwerk-Schwierigkeit (Millisekunden auf Devnet).

Die Optionen `--geo-*`/`--health-*` konfigurieren GeoDNS und
Health-Checking **des eingebetteten DNS-Resolvers**: vollständige
Semantik, Vorrangs-Reihenfolge und die lokale GeoIP-Tabelle sind in
[`dns.md`](dns.md#geodns-geografisches-routing) dokumentiert — das
geografische Routing ist **immer aktiv** (eingebettete
GeoIP-Tabelle, keine Option erforderlich); `--health-check` ist
standardmäßig aus, der klassische Pfad unverändert.

### `domain register|update` (M5) — signierte Transaktionen in einem Befehl

Diese Befehle ketten Bauen → Signieren (Keystore) → Einreichen →
(Bestätigung) in einem einzigen Aufruf.

- **register**: baut ein signiertes `RegisterDomain` (Zeitstempel
  = now, leeres Proof auf Devnet), reicht es ein, wartet auf
  On-Chain-Bestätigung (Sequenz ≥ 0, 30-s-Timeout). Ein bereits
  registrierter Name wird client-seitig UND relay-seitig
  abgelehnt.
- **update**: die Eintragsdatei ist die einzige Quelle der
  Wahrheit. Die Sequenz (on-chain + 1) und der `record_hash`
  (kanonisches BLAKE3 der Einträge der Datei) werden **abgeleitet**,
  nie eingegeben: Signer und Prüfer können nicht divergieren. Nach
  der UpdateDomain-Bestätigung wird der `SignedDnsRecord`
  (Einträge aus der Datei + Owner + Ed25519-Signatur über der
  kanonischen Kodierung) über `put_record` im DHT veröffentlicht
  (das erneut gegen die Chain verifiziert). Eine unregistrierte
  Domain, eine Nicht-Owner-Identität oder ein Dateifehler scheitern
  sauber, ohne die Chain zu berühren.

#### Eintragsdatei

Text, ein Eintrag pro Zeile, `#` = Kommentar, Leerzeilen
ignoriert; die sortierte kanonische Kodierung sorgt dafür, dass die
Reihenfolge der Datei keinerlei Wirkung auf den committeten Hash
hat. Die Datei ist die VOLLSTÄNDIGE Menge (ein Update ersetzt, es
führt nicht zusammen). Typen:

```text
A 192.0.2.1
AAAA 2001:db8::1
CNAME www.example.uip
NS ns1.example.uip
MX 10 mail.example.uip
TXT free text (spaces normalized)
```

Typisierte Fehler: unbekannter Typ, ungültige IP/Name (mit
Zeilennummer), leere Menge, Duplikat, > `MAX_RECORDS_PER_SET`.

## `--genesis` (Founder)

`scone relay --genesis genesis.bin` — **nur Founder-Knoten**
(Kickstarter-Kampagne; die Datei wird von der Binärdatei
`scone-genesis` erzeugt). Normative Regeln:

1. Die Datei wird **vor dem Lesen** abgelehnt, wenn sie 1 MiB
   überschreitet, dann strikt dekodiert (`decode_complete`: keine
   nachfolgenden Bytes). Der Hash von Block 0 wird **neu
   berechnet**, nie aus der Datei gelesen.
2. Semantische Validierung: Höhe 0, `consensus` beginnend mit der
   Netzwerk-Id des angeforderten Netzwerks (`--network`), keine
   Transaktionen, kanonisches Allokations-Manifest. Jede
   Verletzung = typisierter Fehler beim Start (nie eine Panik, der
   Daemon startet nicht).
3. **Jungfräuliches** Datenverzeichnis: die Chain wird auf der
   allokierten Genesis geboren — Allokationen werden im RAM
   angewandt **vor jedem Festplatten-Schreiben**, dann persistiert
   (gesäte Zustände + Genesis-Block unter einem Meta-Schlüssel).
   Der erste produzierte Block kann daher nur die allokierte
   Genesis erweitern.
4. **Bereits produktives** Datenverzeichnis: die Geburts-Genesis
   des Stores ist der `prev_hash` von Block 1
   (selbst-zertifizierend). Gleicher Datei-Hash = stilles OK (die
   Option ist **nach dem ersten Start redundant**); anders =
   typisierte Verweigerung — ein Knoten joint einem Netzwerk über
   Sync, nie über `--genesis`.
5. Neustart **ohne** die Option: ein auf Founder-Art geborenes
   Datenverzeichnis gewinnt seine allokierte Genesis über den
   Meta-Schlüssel zurück (RAM-Rebind bei jedem Boot).

Das Konstante-Genesis-Hash-Pinning (gegen Netzwerk-Divergenz) ist
ein separater Arbeitsstrang.

### Manifest der Binärdatei `scone-genesis` (Kickstarter)

Die Binärdatei `scone-genesis` (Crate `scone-genesis`) bereitet das
Manifest `genesis-manifest.json` (Format `ALLOC-V1`) vor und
produziert dann `genesis.bin`. Normative Regeln:

- `scone-genesis new --network <netzwerk>` erstellt ein leeres
  Manifest; `add`/`remove`/`list` verwalten Allokationen, `build`
  produziert `genesis.bin` (+ Hash), `verify` validiert es neu.
- `add domain <name>` verlangt, dass die **TLD der Domain bereits
  im Manifest allokiert ist** (sonst sofortige Zurückweisung — die
  Chain hätte sie bei der Genesis-Anwendung verweigert); ihr
  `valid_until` ist **automatisch**: `born_at + 1 Jahr` (die
  default_validity des Manifests), nie von Hand eingegeben.
- `add tld <name>`: **standardmäßig offen** (niemands,
  Self-Service, 1-Jahres-Untergrenze, Owner = null auf dem Wire —
  Art `0x03`); `--private` = privat (Owner, Genesis-Laufzeit
  `born_at + TLD_TERM_SECS` — 3 Jahre, Standard-Lebenszyklus:
  `RenewTld`, 60-Tage-Karenz).
- Kanonische Sortierung nach Name (gleiche Inhalte → gleicher
  Block → gleicher Hash, egal in welcher Reihenfolge eingefügt);
  Duplikate verweigert.

## Grenzen und Speicher-Garantien

| Ressource | Grenze |
|---|---|
| P2P-Frame | `MAX_MESSAGE_LEN` (1 MiB) vor dem Parsen |
| Mempool | 4096 Tx (darüber typisierte Zurückweisung); `MAX_PENDING_PER_DOMAIN = 4` ausstehende Txs pro Domain |
| Mempool-Gnade | `MAX_GRACE_ATTEMPTS = 10` Wiedereinfügungen einer aus nicht-definitivem Grund gespülten Tx (≈ 7 Min bei 40 s/Zyklus), dann Verdrängung |
| Fork-Parking | `PARKED_CAP = 2048` konkurrierende Blöcke |
| Akkumulierte Checkpoints | `MAX_PENDING_CHECKPOINTS = 256` (Verdrängung: kleinster Daten-Hash) |
| Gleichzeitige DHT-Wartende | `MAX_DHT_WAITERS` (256); DHT-Nachschlag auf 30 s begrenzt (`DHT_LOOKUP_TIMEOUT`) |
| Blöcke pro Sync-Antwort | 1 pro Roundtrip; ≤ 10.000 Stapel pro Peer (`MAX_SYNC_ROUNDS`); Protokollgrenze `MAX_BLOCKS_PER_REQUEST = 128` auf der Anfrage |
| DHT-Eintrag | `MAX_DHT_CACHE_ENTRY` (64 KiB) |
| Persistenter DHT-Cache (Festplatte) | doppelte LRU-Grenze: `DHT_CACHE_MAX_ENTRIES` (100.000 Einträge) UND `DHT_CACHE_MAX_BYTES` (2 GiB) — welche zuerst trifft, verdrängt nach Schlüssel-Reihenfolge; GC-Verdrängung in DERSELBEN Transaktion wie der Block (eine von der Chain-GC getötete Domain hinterlässt nie einen Waisen-Eintrag) |
| DHT-Negativcache | `DHT_NEGATIVE_TTL_SECS` (30 s), FIFO-Cap `DHT_NEGATIVE_CAP` (4.096) |
| Chain im RAM | Genesis + Spitz-Fenster (Historie aus dem Store bedient) |

Anti-Schleifen-Regel (H2): eine Transaktion wird nur neu gesendet,
wenn sie **tatsächlich** in den Mempool eingefügt wurde (erster
Anblick); ein Duplikat (bereits im Pool oder bereits in der Chain)
wird nie neu gerelayt. Ein bereits kanonischer Block wird erkannt
und vor `push_block` ignoriert (kein Neu-Speichern, kein
Neu-Broadcast).

Offene Entscheidungen: persistente Netzwerk-Identität des Relays,
Broadcast-Gossip (aktuell: Unicast-Request-Response an jeden
bekannten Peer). Seit dem ursprünglichen Schreiben gelöst: echte
Fork-Wahl (Parking + reconsider_parked, siehe oben), PoS-Konsens
(signierter Produzent + Anker-Komitee), DNS-Resolver (die
`--dns`-Fläche, siehe dns.md), Steuerkanal
(RPC→p2p-Vereinheitlichung: kein JSON-TCP mehr).

## Grazielles Herunterfahren (SIGTERM / SIGINT)

Der Daemon `scone relay` (Vordergrund) fängt SIGTERM und SIGINT
über `tokio::signal`: beim Signal wird die `Relay::run`-Future
abgebrochen (`tokio::select!` in der Binärdatei), was den
libp2p-Schwarm fallen lässt und die QUIC-Sockets schließt. Die
tokio-Laufzeit wird dann mit einem begrenzten Herunterfahren (≤ 2 s)
gestoppt, das die losgelösten DNS-Tasks (UDP/TCP) fallen lässt und
ihre Sockets schließt — das Ziel ist, dass ein `kill` keinen
verwaisten UDP-Socket hinterlässt (Fall beobachtet auf Testnet 60).
Ein signal-getriebenes Herunterfahren loggt `relay shutting down
(signal)` (INFO, stderr) und beendet mit Exit-Code **0**: ein
gewünschtes Herunterfahren ist kein Fehlschlag. Falls die
Handler-Installation scheiterte, gilt das Standardverhalten des
Signals (Degradation, nie eine Panik).


## Tor-Transport (offene Frage)

QUIC/UDP durchdringt keine einfachen SOCKS-Proxys, daher können
Relays heute nicht trivial über Tor laufen. Ein nativer
libp2p-Tor-Transport (Relay mit Onion-Adresse) ist der verfolgte
Weg nach vorn — siehe die Zensurresistenz-Anleitung der Website für
den aktuellen Stand und die `.tor`-CNAME-Brücke (lesbare Namen für
Hidden Services, heute unterstützt).

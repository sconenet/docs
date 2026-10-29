# Speicherung — lokale Knoten-Persistenz (redb)

> Normatives Dokument. Jede Abweichung zwischen diesem Dokument und
> `crates/scone-storage/src/` ist ein Bug (die Quelle der Wahrheit
> ermitteln und beheben). Implementationsreferenz: Crate
> `scone-storage` (Format v1), `redb` 4.2-Backend.

## Zweck

Die eingebettete lokale Persistenz des Knotens: kanonische Blöcke,
Domain-Zustände, DHT-Cache, Metadaten. Der Store ist eine
**Persistenz**, nie eine Autorität: der RAM-Zustand
([`ChainState`], die Konsensquelle) wird beim Start aus dem Store
rekonstruiert, und jedes Schreiben ist ein **Delta** (von einem
Block modifizierte Domains), nie eine vollständige Kopie des
Zustands.

Querschnitts-Zwang (nutzergewandt, jeden Meilenstein): **minimaler
Speicher-Fußabdruck**. Große Volumina (Blöcke, Domain-Zustände,
DHT-Cache) leben in redb und werden über Cursor/Bereiche
zugegriffen — nie vollständig in RAM geladen, nie nutzlos
dupliziert.

## Grenze: der Trait `NodeStore`

`crates/scone-storage/src/lib.rs` definiert die einzige
Speicherungs-Schnittstelle, die für den Rest des Codes sichtbar
ist (Blockchain, künftiges M4-Relay):

- `append_block(height, hash, bytes)` /
  `append_block_with_state(..., deltas)` — atomares Anhängen (siehe
  unten);
- `block_at_height(h)` / `block_by_hash(hash)` — Punkt-Lesungen;
- `tip() -> (height, hash)` — O(1), aus `meta` gelesen;
- `put_domain_state` / `domain_state` / `domain_count` —
  **gepflegter** Zähler (`meta["domain_count"]`), nie ein
  `COUNT(*)`;
- `iterate_domains(after, max) -> DomainPage` — cursor-basierte
  Paginierung (standardmäßig Stapel von 100), aufsteigende
  Byte-Reihenfolge der `DomainId`;
- `put_tld_state` / `tld_state` / `tld_count` / `iterate_tlds` —
  dieselben Verträge für das TLD-Register (`tld_count` gepflegt,
  begrenzter Cursor) (M7d);
- `put_dht_cache` / `dht_cache` — opake `SignedDnsRecord`-Bytes;
- `resolve_name(canonical)` / `resolve_tld(tld)` — persistenter
  Name→Id-Index (P0.2), O(1)-Punkt-Nachschlag, keine Iteration über
  Domains (Standard `None` = Backend ohne Index, der Aufrufer fällt
  auf `iterate_domains` oder den Chain-Zustand zurück);
- `tld_floors_all` / `snapshot_tld_floors_all` — Lesen der
  Lebenszeit-Untergrenzen offener TLDs (Owner-Option: dedizierte
  Tabellen, lebend `tld_floors_v1` und eingefroren
  `snapshot_v1_floors`);
- `rewind_to_height(h)` — Reorg: entfernt die Blöcke über `h` und
  bringt `tip`/`tip_height` in EINER Transaktion zurück auf `h`
  (Vorstufe zu `persist_state_snapshot`, siehe unten);
- `meta_get` / `meta_set`;
- `snapshot_meta` / `snapshot_domains` / `snapshot_tlds` — Lesen
  des Boot-Zustands-Snapshots (M6b; leere Standards = Backend ohne
  Snapshot, `load_chain` fällt dann auf volles Laden zurück).

Regeln:

- **Keine redb-Typen in der API**: nur Bytes (`&[u8]`, `[u8; 32]`)
  und Core-Typen. Das Backend ist ersetzbar.
- **Async-frei, keine innere Veränderlichkeit**: Schreibvorgänge
  nehmen `&mut self` (redb serialisiert Schreiber), Lesevorgänge
  `&self` (eine kurze Lese-Transaktion pro Aufruf).
- Die API validiert Blöcke nie neu: der an `append_block*`
  übergebene Hash ist der vom Aufrufer **neu berechnete** Hash
  (`Blockchain::push_block`).

## redb-Tabellenschema

| Tabelle | Schlüssel (Sortierung) | Wert |
|---|---|---|
| `blocks_by_height` | `u64` Höhe (geordneter B-Baum) | `hash(32) ‖ height(8 BE) ‖ kanonischer Block` |
| `blocks_by_hash` | Block-Hash (32 B) | kanonische Kodierung des Blocks |
| `domains` | `DomainId` (32 B, Byte-Reihenfolge) | `DomainStateBytes` (unten) |
| `tlds` | `TldId` (32 B, Byte-Reihenfolge) | `TldStateBytes` (unten, M7d) |
| `dht_cache` | `DomainId` (32 B) | `SignedDnsRecord`-Bytes (opak) |
| `names_v3` (P0.2) | Index-Schlüssel (32 B, Byte-Reihenfolge) | Ziel-Id (32 B) |
| `name_reverse_v3` (P0.2) | Ziel-Id (32 B, Byte-Reihenfolge) | `tag(1) ‖ kanonischer Name` |
| `meta` | `&[u8]` | `&[u8]` |
| `snapshot_v1_meta` (M6b) | `u64` = 0 (einziger Slot) | `height(8 BE) ‖ tip_hash(32)` = genau 40 B |
| `snapshot_v1_domains` (M6b) | `DomainId` (32 B, Byte-Reihenfolge) | `DomainStateBytes` (dasselbe Format wie `domains`) |
| `snapshot_v1_tlds` (M6b) | `TldId` (32 B, Byte-Reihenfolge) | `TldStateBytes` (dasselbe Format wie `tlds`) |
| `tld_floors_v1` (Owner-Option) | `TldId` (32 B, Byte-Reihenfolge) | `u64` (Ablauf-Untergrenze, redb implizit BE) |
| `snapshot_v1_floors` (M6b) | `TldId` (32 B, Byte-Reihenfolge) | `u64` — eingefrorene Kopie der Untergrenzen (Zustand BEIM Snapshot, nicht die Spitze) |

Zusätzlicher `meta`-Schlüssel (P0.3/P0.4):

| Schlüssel | Wert |
|---|---|
| `verified_snapshot_v1` (P0.3) | `height(8 BE) ‖ tip_hash(32) ‖ manifest_hash(32)` = 72 B — Marker eines VERIFIZIERTEN importierten Snapshots (reserviert, wie `tip`) |

`meta`-Schlüssel:

| Schlüssel | Wert |
|---|---|
| `format_version` | `u64 BE` = `STORAGE_FORMAT_VERSION` (1) |
| `tip` | Hash (32 B) des Spitz-Blocks |
| `tip_height` | `u64 BE` (macht `tip()` O(1)) |
| `domain_count` | `u64 BE` (gepflegter Zähler) |
| `tld_count` | `u64 BE` (gepflegter Zähler, M7d) |

Der `hash ‖ height`-Header von `blocks_by_height` macht jeden Index
in sich geschlossen (verifizierbar ohne den Block zu lesen); beide
Indizes werden in derselben Transaktion aktualisiert.

### Genesis-Konvention

Der Genesis-Block ist deterministisch (Protokollkonstanten) und
wird **nie** als Bytes gespeichert. Ein frischer Store hat
`tip == (0, genesis_hash)`; der zuerst angehängte Block hat Höhe 1.
Der Genesis-Hash wird immer neu abgeleitet
(`scone_blockchain::genesis_hash()`), nie gelesen.

## Kodierungsformat `DomainStateBytes` (M8b)

65 oder 97 Bytes je nach führendem Versions-Tag, fest-breite
Big-Endian-Ganzzahlen (kanonisch, Byte für Byte vergleichbar):

```text
DomainStateBytes = tag ‖ owner(32) ‖ sequence(8 BE) ‖ registered_at(8 BE)
                   ‖ valid_until(8 BE) ‖ mutations(8 BE) [‖ record_hash(32)]

tag = 0x01 : record_hash vorhanden → exakt 97 Bytes
tag = 0x02 : record_hash abwesend  → exakt 65 Bytes
```

- `registered_at` / `valid_until` (M8b): der Ablauf der
  Registrierung (1 Jahr beim Claim, erneuerbar, siehe
  `/docs/en/technical/blockchain.md`).
- `mutations` (V3): der monotone Mutationszähler des Namespace
  (CAS-Anti-Replay-Generation — siehe `transactions.md`, Der
  Mutationszähler). Die M8b-Kodierungen (57/89 Bytes, ohne Zähler)
  sind **illegitim**: striktes Dekodieren → `Corrupted` (Testnet-
  Reset; ein Prä-Zähler-Store regeneriert sich via Replay).

Die M7-Kodierungen (41/73 Bytes, kein Ablauf) sind seit M8b
**illegitim**: striktes Dekodieren → `Corrupted`. (Ein M7-Store
regeneriert über Replay: `load_chain_replay`.)

- `owner`: `OwnerId` (32 B);
- `sequence`: die zuletzt angewandte `UpdateDomain`-Sequenz (0 gleich
  nach einem `RegisterDomain`);
- `record_hash`: das aktuelle On-Chain-Commitment des
  DNS-Eintragssatzes (`None`, solange kein `UpdateDomain` angewandt
  wurde).

**Strenges** Dekodieren (`DomainStateBytes::decode`): unbekanntes
Tag, falsche Länge oder überschüssige Bytes →
`StorageError::Corrupted`, nie eine Panik.

## Kodierungsformat `TldStateBytes` (Owner-Option)

Variable Länge je nach Enum-Variante — 50 Bytes für eine
**private** TLD, 10 für eine **offene** TLD (weder Sequenz noch
Eintrags-Hash):

```text
TldStateBytes = tag(0x04) ‖ flag(0x00|0x01) ‖ owner(32)? ‖ until(8)
                [‖ mutations(8)]
```

- `flag` (Owner-Option, Enum `TldState`): `0x00` =
  `TldState::Open` (niemands — Self-Service + PoW, keine
  Owner-Bytes), `0x01` = `TldState::Private` (32 Owner-Bytes
  folgen). Streng kanonisch — jedes andere Byte → `Corrupted`. Die
  Invariante „offen ⟺ kein Owner“ ist STRUKTURELL: die zwei
  Zustände können nicht verwechselt werden.
- **`until` (8 Bytes, BE64)**: die eigene Uhr der TLD — das Ende
  der privaten Laufzeit (Claim/Verlängerung + 3 Jahre) oder die
  offene Todesplanung (1-Jahres-Untergrenze bei Geburt, von jeder
  lebenden Domain geschoben). Es ist wieder ein Attribut des
  Eintrags (TLD-Lebenszyklus-Änderung). Die abgeleitete Tabelle
  `tld_floors_v1` (+ ihre eingefrorene Kopie `snapshot_v1_floors`)
  verfolgt weiterhin die Untergrenzen, beim Replay neu gebaut und
  beim Boot wiederhergestellt — ein neu gestarteter Knoten tötet
  dieselben TLDs wie ein kontinuierlicher.

Die Prä-Zähler-Layouts Tag `0x04` (42/2 Bytes, ohne Zähler), die
Kodierungen M7d (33 Bytes, Tag `0x01`), M8b (34 Bytes, Tag
`0x02`, boolesches `open`) und das Alt-Tag `0x03` (eingebettetes
`until`) sind **illegitim**: striktes Dekodieren → `Corrupted` (vor
dem Mainnet ist Regeneration über Replay der
Migrationspfad).

Das Tag existiert nur für die Format-Entwicklung: ein künftiges
Layout erhöht das Tag, und alte Leser scheitern mit `Corrupted`,
statt zu raten. **Strenges** Dekodieren
(`TldStateBytes::decode`): unbekanntes Tag, falsche Länge,
überschüssige Bytes oder nicht-kanonisches Flag →
`StorageError::Corrupted`, nie eine Panik.

## Atomaritätsgarantien

`append_block_with_state` schreibt in **einer einzigen** redb
`WriteTransaction`:

1. den Block in `blocks_by_height` (Vorab-Prüfung: die Höhe darf
   nicht mit anderem Inhalt existieren, sonst
   `StorageError::Conflict`);
2. den Block in `blocks_by_hash`;
3. jedes Domain-Zustands-Delta (`domains`);
4. den Zähler `domain_count` (nur um die Anzahl **neuer** Domains
   erhöht; ein `UpdateDomain` einer bestehenden Domain erhöht ihn
   nicht);
5. jedes TLD-Register-Delta (`tlds`, M8b: Claims, Transfers,
   öffnen/schließen) — **in derselben Transaktion**: ein Block ohne
   seine TLD-Änderungen ist strukturell unmöglich;
6. den Zähler `tld_count` (nur neue TLDs);
7. jede **Entfernung** (M8b): durch die deterministische GC
   abgelaufene Domains (`removed_domains`) und widerrufene TLDs
   (`removed_tlds`) verlassen den Store in derselben Transaktion,
   Zähler dekrementiert — ein Neustart kann eine abgelaufene
   Registrierung nie auferstehen lassen;
8. die **Namensindex**-Deltas von P0.2 (Upserts und Entfernungen,
   siehe unten) — dieselben Garantien: ein Absturz hinterlässt nie
   einen Index-Eintrag, der auf eine fehlende Domain zeigt;
9. `tip` und `tip_height`.

redb committet WAL-artig: ein Absturz mitten im Schreiben hinterlässt
den **vorherigen** kohärenten Zustand. **Ein Block ohne seinen
Zustand zu schreiben ist daher strukturell unmöglich** — das ist
die Schlüsselinvariante des Meilensteins.

Zusätzliche Garantien:

- **Monotonie**: Anhängen auf einer Höhe ≠ `tip_height + 1` →
  `StorageError::NonMonotonicHeight`, es wird nichts geschrieben
  (geprüft *vor* dem Öffnen der Schreib-Transaktion);
- **Idempotenz**: erneutes Anhängen des aktuellen Spitz-Blocks mit
  demselben Paar `(height, hash)` ist ein No-Op (der schnelle Pfad
  vergleicht Höhe und Hash; bei gleicher Höhe sind identische
  Block-Bytes ebenfalls ein stilles No-Op, andere Bytes ein
  `Conflict`);
- **Korrumpierte Datenbank**: abgeschnittene/verunreinigte Datei →
  typisierter Fehler (`StorageError::Database`/`Corrupted`) beim
  Öffnen oder Lesen, nie eine Panik (getestet);
- **Formatversion**: eine `format_version` ≠ 1 →
  `StorageError::UnsupportedFormat` (ein expliziter
  Migrationspfad ist erforderlich, nie Raten).

## Blockchain-Integration (`integration.rs`)

Rollentrennung: [`ChainState`] im RAM = Konsensautorität;
Store = Persistenz. Die gewählte Strategie (die einfachste, wie
dokumentiert): **Zustand bei jedem Block persistieren** —
`store_block` schreibt den Block + die Zustände der von diesem
Block **berührten** Domains (Delta). Vollständiges Replay bleibt
möglich und dient als Kontroll-/Reparaturpfad.

- `store_block(store, chain, block, hash)`: kodiert den Block
  kanonisch, liest den Endzustand der berührten Domains UND der
  beanspruchten TLDs aus dem RAM-Zustand (bereits von `push_block`
  aktualisiert), atomares Anhängen.
- `load_chain(store)`: **ohne Replay** — liest Spitze + Spitz-Block
  (O(1)-Block-Lesungen), stellt den Domain-Zustand in Seiten von
  100 wieder her (`DOMAIN_PAGE`) **und das TLD-Register in Seiten
  von 100** (`TLD_PAGE`, M7d — Schließung des Neustart-Fensters:
  ohne ihn vergaß ein neu gestarteter Knoten jede beanspruchte TLD
  und lehnte jede `RegisterDomain` unter dieser TLD mit
  `UnknownTld` ab, obwohl seine Festplatte den vollständigen
  Zustand hielt). Die RAM-Chain behält nur Genesis + Spitze;
  historische Blöcke bleiben im Store und werden von dort bedient.
  Seit M8 baut `load_chain` dann den **TXID-Anti-Replay-Index**
  neu auf (siehe unten).
- `load_chain_replay(store)`: spielt alle gespeicherten Blöcke
  einzeln ab (nie alle im RAM) mit vollständiger Validierung; die
  resultierende Spitze muss der gespeicherten gleichen, sonst
  `Corrupted`.
- `rebuild_replay_index_from_store(store, chain, tip_height)`
  (M8): scannt die letzten
  `min(REPLAY_WINDOW_BLOCKS, tip_height)` persistierten Blöcke (ein
  Block gleichzeitig im RAM) neu und fügt jede TxId **auf ihrer
  ursprünglichen Einschlusshöhe** in den Index der Chain wieder
  ein, über die öffentliche API `Blockchain::rebuild_replay_index`
  (kein Zugriff auf private Felder). Fehlender Block unter der
  Spitze oder nicht dekodierbar innerhalb des Fensters →
  `Corrupted` (diese Bytes dienten bereits zum Aufbau der Chain:
  ein Fehlschlag hier signalisiert einen beschädigten Store).
- `touched_domains(block)`: Ids der von den Transaktionen des
  Blocks modifizierten Domains, dedupliziert, Blockreihenfolge.
- `touched_tlds(block)`: Ids der vom Block mutierten TLDs
  (`RegisterTld`, `TransferTld`, `RevokeTld`, `SetTldOpen`),
  dedupliziert, Blockreihenfolge (M8b).
- `store_block_with_removals(...)`: Variante von `store_block`, die
  zusätzlich die vom Block-GC entfernten Domains erhält (M8b) für
  das oben beschriebene atomare Schreiben. Das Relay speist diese
  Liste aus `Blockchain::push_block_with_gc` (das
  Anwendungsergebnis legt `gc_removed_domains` offen). **GC-
  Verdrängung des DHT-Caches**: die `dht_cache`-Einträge
  entfernter Domains werden in DERSELBEN Transaktion wie der Block
  gelöscht (Anhang-Deltas tragen die Tabelle `dht_cache`) — ein
  DHT-Eintrag überlebt nie die Domain, der er diente, abgelaufene
  Einträge akkumulieren sich nie für immer (Test:
  `gc_evicts_the_dht_cache_entry`).

Das Relay (M4) wird verwenden: `load_chain` beim Start, dann für
jeden akzeptierten Block `push_block` (RAM) dann `store_block`
(Festplatte).

## Reorg-Persistenz (`rewind_to_height` + `persist_state_snapshot`)

Ein Reorg (Wechsel auf einen stärkeren Schwesterzweig) kann nicht
über das monotone Anhängen laufen — das Schreiben des
gewinnenden Suffixes würde mit `NonMonotonicHeight` scheitern. Der
dedizierte Pfad (Sticky-Minority-Fix, Relay + Store):

1. `rewind_to_height(fork_height)` — EINE redb-Transaktion entfernt
   die Blöcke über dem Fork-Punkt und bringt `tip`/`tip_height`
   zurück (der Namensindex und die lebenden Zustände werden im
   nächsten Schritt korrigiert);
2. `persist_state_snapshot(store, chain)` — schreibt die lebenden
   Tabellen (`domains`, `tlds`, Namensindex, TLD-Untergrenzen) aus
   dem Post-Reorg-RAM-Zustand-Export neu: die lebenden Tabellen
   beschreiben beim Boot NIE den VERLIERENDEN Zweig;
3. monotones erneutes Anhängen des adoptierten Suffixes
   (gewöhnliche Deltas).

Ein Persistenz-Fehlschlag auf diesem Pfad wird geloggt und
degradiert (der Store heilt beim Boot über Replay) — nie fatal.

## Boot-Zustands-Snapshot (M6b)

`load_chain` wie oben beschrieben ist immer noch O(Größe des
Zustands): kein Block wird abgespielt, aber die gesamte Tabelle
`domains` wird neu gelesen. Es ist der Zustand-pro-Block-persistiert-Pfad,
der Replay unnötig macht. Der M6b-Snapshot adressiert andere
Kosten: den Fall, in dem der RAM-Zustand **durch Replay**
rekonstruiert werden muss (Reparatur, Kontrolle oder ein Backend
ohne pro Block persistierten Zustand). Er bietet einen Boot in
**O(tip − H)** — Abspielen nur des Suffixes über dem Snapshot.

### Schreiben

- **Intervall**: `SNAPSHOT_INTERVAL = 64` (öffentliche Konstante
  von `scone-storage`). Der Auslöser ist das Anhängen von Block
  `k·SNAPSHOT_INTERVAL + 1`: diese Transaktion friert den Zustand
  der lebenden Tabellen **ein, wie er vor der Anwendung dieses
  Blocks ist**, d. h. den kanonischen Zustand nach Block
  `k·SNAPSHOT_INTERVAL`.
- **Nie an der Spitze**: der Snapshot liegt also höchstens bei
  `tip − 1`, zum Schreibzeitpunkt immer strikt unter der Spitze.
  Ein Absturz direkt danach kann nie einen Snapshot „über“ der
  persistierten Chain hinterlassen.
- **Atomar**: das Einfrieren (Leeren + Kopieren der Tabellen
  `domains`/`tlds` nach `snapshot_v1_*`, Schreiben von
  `snapshot_v1_meta`) geschieht in DERSELBEN redb-Transaktion wie
  das Anhängen des Auslöser-Blocks — ein Absturz hinterlässt
  entweder den alten Snapshot intakt oder den neuen vollständig
  mit seinem Block (redb WAL).
- **Einziger Slot**: jeder Snapshot ERSETZT den vorherigen
  (Tabellen geleert, dann neu kopiert) — keine Akkumulation, kein
  zu verwaltendes Beschneiden, der Festplatten-Fußabdruck ist
  durch ~2× des Zustands begrenzt.
- Das Format der eingefrorenen Zustände ist strikt das der
  `DomainStateBytes`/`TldStateBytes` (Wiederverwendung, keine neue
  Kodierung); nur die Zeile `snapshot_v1_meta` ist ein neues Format
  (`height(8 BE) ‖ tip_hash(32)`, striktes Dekodieren: jede andere
  Länge → `Corrupted`).

### Lesen beim Boot (`load_chain`)

1. Die Spitze wird wie immer validiert (Hash aus dem Header des
   Spitz-Blocks neu berechnet gegen `meta["tip"]`).
2. Wenn `snapshot_meta()` existiert, seine Höhe `H` die Bedingung
   `0 < H < tip_height` erfüllt und der NEU BERECHNETE Hash des auf
   Höhe `H` gespeicherten Blocks dem `tip_hash` des Snapshots
   gleicht (**Anker** — ein Snapshot, der fremd zur persistierten
   Chain ist, verändert oder aus einer anderen Datei, wird hier
   erkannt), wird der eingefrorene Zustand in Seiten von 100
   wiederhergestellt, dann werden nur die Blöcke `H+1..=tip`
   abgespielt (vollständige Validierung, identisch lebenden
   Blöcken).
3. Die durch Replay erreichte Spitze muss der gespeicherten
   gleichen.
4. **Jeder Fehlschlag auf diesem Pfad (fehlender Snapshot,
   unbrauchbare Höhe, ungültiger Anker, nicht dekodierbare Bytes,
   gescheitertes Replay) degradiert still zum vollständigen Laden
   vor M6b** (Lesen der lebenden Tabellen, ohne Replay) — ein
   schlechter Snapshot lässt den Boot nie scheitern. Die
   Rückwärtskompatibilität ist der Sonderfall „kein Snapshot“: ein
   vor M6b geschriebener Store (ohne die `snapshot_v1_*`-Tabellen)
   lädt identisch, da `snapshot_meta()` `None` ist.

Das abgespielte Suffix ist durch `SNAPSHOT_INTERVAL` begrenzt
(höchstens 64 Blöcke zum Auslöser-Zeitpunkt + die seither geminten
Blöcke): der Boot ist O(tip − H) mit garantiert rezentem H.

## Beim Boot neu aufgebauter TXID-Anti-Replay-Index (M8)

Der Anti-Replay-Index (`included`, TxId → Einschlusshöhe, Fenster
`REPLAY_WINDOW_BLOCKS = 256` — siehe
`/docs/en/technical/blockchain.md`) ist eine **reine Funktion der
kanonischen Chain**: er wird nie persistiert. Da `load_chain` die
Historie nicht abspielt, startete eine vor M8 wiederhergestellte
Chain mit **leerem** Index — ein neu gestarteter Knoten akzeptierte
die Wiedereinbindung jeder Transaktion, die noch in ihrem
Replay-Fenster war (dokumentierte Einschränkung; die Zustandsregeln
blieben die einzige Absicherung).

M8 schließt dieses Fenster: nach der Wiederherstellung (voller
Pfad ODER Snapshot + Suffix-Replay) ruft `load_chain`
`rebuild_replay_index_from_store` auf, das die letzten
`min(REPLAY_WINDOW_BLOCKS, tip_height)` persistierten Blöcke neu
liest — ein Block gleichzeitig im RAM, wie überall
speicherbegrenzt — und jede TxId auf ihrer **ursprünglichen Höhe**
wieder einfügt. Ein Store mit weniger als 256 Blöcken baut also
seinen gesamten Index neu auf.

Eigenschaften:

- **Entscheidungen identisch zum lebenden Knoten**: dieselben
  Einträge, dieselbe Beschneidungsregel (`h > tip − 256`) — ein
  Knoten, der neu startet, und ein Knoten, der nie anhielt, lehnen
  dieselben Replays ab (getestet: exakte Index-Anzahl und
  Gleichheit mit dem beschchnittenen lebenden Index).
- **Unveränderte Beschneidungs-Semantik**: eine Transaktion älter
  als das Fenster bleibt vom Index zugelassen (und scheitert dann
  ggf. an den Zustandsregeln) — exakt wie zuvor.
- **Der Snapshot friert den Index NICHT ein**: das Suffix-Replay
  des Snapshots rekonstruiert die Einträge über H, der M8-Rescan
  fügt den Teil des Fensters UNTER H hinzu (derselbe Vertrag wie
  owner_keys/Karenz: neu gebaut, nie persistiert).
- **Kosten**: ≤ 256 Block-Punkt-Lesungen beim Boot, O(Fenster) —
  unabhängig vom Alter der Chain.
- Ein fehlender oder nicht dekodierbarer Block **innerhalb des
  Fensters** lässt den Boot mit `Corrupted` scheitern (diese Bytes
  dienten bereits zum Aufbau der Chain); außerhalb des Fensters
  werden historische Blöcke nicht berührt.

`load_chain_replay` (Null-Vertrauen) baut den Index auf demselben
Weg über sein volles Replay neu auf — beide Pfade konvergieren.

## Persistenter Name → Id Index (P0.2)

`DomainId = BLAKE3-256(DOMAIN_ID_VERSION ‖ kanonischer Name)` ist
nicht invertierbar: die Auflösung eines Namens (Relay/DNS)
erfordert einen persistenten Index. P0.2 fügt ihn dem Store hinzu,
mit derselben transaktionalen Disziplin wie der Rest.

### Schema

Zwei Begleit-Tabellen, zusammen geschrieben und gelöscht:

```text
names_v3        key  = BLAKE3-256("SCONE-NAME-IDX-V1" ‖ kanonischer Name)  (32 B)
                val  = Ziel-Id (32 B : DomainId oder TldId)

name_reverse_v3 key  = Ziel-Id (32 B)
                val  = tag(0x01 Domain | 0x02 TLD) ‖ kanonischer Name
```

- Der Schlüssel `names_v3` ist ein **Hash des Namens, nicht der
  Klartextname** (kompakte Index-Tabellen, kein Name-Gossip über
  Dumps). Der Schlüssel ist eine **reine Funktion des Namens**: der
  Index ist jederzeit aus der Chain neu aufbaubar
  (`rebuild_name_index`), für ihn werden keine zusätzlichen
  Metadaten persistiert.
- `name_reverse_v3` speichert den Namen (genau einmal), weil die
  Entfernungs-Pfade (GC abgelaufener Domains, `RevokeTld`) nur
  **Ids** kennen: der zu löschende Hash-Schlüssel muss irgendwie
  gefunden werden. Das Tag unterscheidet die zwei Namespaces.
- Strenge Trennung: `resolve_name` antwortet NIE mit einem
  TLD-Eintrag und `resolve_tld` nie mit einem Domain-Eintrag (das
  Tag der umgekehrten Tabelle entscheidet; ein `DomainName` hat
  ≥ 2 Labels, eine TLD hat 1 — beide Schlüsselableitungen teilen
  den Raum `names_v3`, aber nicht die Ergebnisse).
- Die `DomainId` bleibt 32 Bytes, nie reduziert (harte
  Projektregel: Konsens-Identität, Kollisionsresistenz).

### Transaktionale Konsistenz

Upserts (`RegisterDomain`, `AssignDomain`, `RegisterTld` — von den
Transaktionen im Klartext getragene Namen) und Entfernungen (GC,
Widerrufe — nackte Ids) werden vom `StateDelta` des Blocks getragen
(`name_upserts` / `name_removals`) und in **derselben redb
Transaktion** wie die Block- und Zustandsschreibungen angewandt.
Ein Absturz kann daher weder einen Index-Eintrag hinterlassen, der
auf eine fehlende Domain zeigt (keine Geister), noch eine
registrierte Domain ohne ihren Eintrag. Ein gescheitertes Anhängen
schreibt nichts (getestet). Die benannten Varianten
`put_domain_state_with_name` / `put_tld_state_with_name` /
`remove_domain_state` bieten Reparatur-Werkzeugen dieselbe
Atomarität.

`store_block` / `store_block_with_removals` leiten die
Namens-Deltas aus DENSELBEN Transaktionen des Blocks ab — kein
zusätzlicher Parameter an der Relay-Schnittstelle.

### Neuaufbau (`RedbStore::rebuild_name_index`)

Der Index ist eine reine Funktion der kanonischen Chain, wie der
M8-Anti-Replay-Index. `rebuild_name_index` baut ihn aus den
**persistierten Blöcken** neu auf (die `DomainStateBytes`
enthalten NICHT den Namen — nur Owner/Sequenz/Ablauf —, also ist
die Quelle der Wahrheit der Transaktions-Strom, wo `RegisterDomain`
und `AssignDomain` den kanonischen Namen im Klartext tragen):

1. Durchgang 1: `blocks_by_height` Genesis→Spitze scannen, ein
   Block gleichzeitig im RAM; die in
   `RegisterDomain`/`AssignDomain`/`RegisterTld` gesehenen
   Bindungen `Name → Id` sammeln;
2. Durchgang 2: auf nur **lebende** Ziele filtern (der
   persistierte Zustand ist autoritativ) — eine registrierte und
   dann per GC/abgelaufene Domain wird nicht neu indiziert;
3. Durchgang 3: Löschen + vollständiges Neuschreiben beider
   Tabellen in EINER Transaktion (Absturz → alter oder neuer
   vollständiger Index).

Ein fehlender oder nicht dekodierbarer Block → `Corrupted`
(dieselben Bytes, die die Chain gebaut haben). Die Kosten sind
O(gesehene verschiedene Namen) im RAM (≤ 253 B pro Name) und
O(Blöcke) an Lesungen — ein Reparatur-Werkzeug, kein Boot-Pfad.

### Was der Index NICHT ist

- Keine Autorität: der RAM-Chain-Zustand bleibt die einzige
  Autorität; der Index ist ein neu aufbaubarer Cache (eine
  Abweichung ist ein Transaktionalitäts-Bug, erkennbar durch
  `rebuild_name_index` im Vergleich dazu).
- Nicht im M6b-Snapshot eingefroren: die Tabellen
  `names_v3`/`name_reverse_v3` sind lebendig und folgen den Deltas
  jedes Blocks; der Zustands-Snapshot bettet sie nicht ein
  (Rekonstruktion über Deltas bei Wiederaufnahme, oder
  `rebuild_name_index` als Reparatur).

### Was der Snapshot NICHT committet

Der RAM-Zustand `ChainState` trägt interne Strukturen, die nicht
vom Konsens committet werden (Owner-Schlüssel, Karenzfenster). Sie
werden NICHT eingefroren: das Suffix-Replay rekonstruiert sie
durch Anwenden der Transaktionen (deterministisch, derselbe
Inhalt wie das vollständige Replay für das relevante Fenster). Der
TXID-Anti-Replay-Index folgt demselben Vertrag (M8, siehe
vorheriger Abschnitt): beim Boot neu gebaut, nie eingefroren.
Bit-exakte Gleichheit mit `load_chain_replay` ist per Test auf dem
kanonischen Zustand verifiziert (state_root + SMT).

## VERIFIZIERTER Zustands-Snapshot — Chunks + Manifest + kryptografische Verifikation (P0.3)

Der obige M6b-Snapshot beschleunigt den Boot eines Knotens, der
seiner Festplatte vertraut. P0.3 adressiert ein anderes Bedürfnis:
einen Zustand zwischen Knoten zu transportieren (Bootstrapping
eines neuen Knotens, Reparatur), ohne dem Transport zu vertrauen.
Das Format und die Verifikation leben in
`crates/scone-storage/src/verified_snapshot.rs` + die
Wurzel-Rekonstruktion in
`crates/scone-blockchain/src/snapshot_verify.rs`.

### Vertrauensmodell

Der Snapshot trägt den Zustand eines **finalisierten** Blocks —
ein vom Komitee-Quorum signierter Checkpoint
(`CheckpointData.state_root`, die O(1)-`state_root_smt`-Wurzel).
Die Authentizität des Checkpoints (Quorum von
Ed25519-Signaturen) wird von der Chain-Schicht VOR dem Import
validiert. Der Import selbst verifiziert die Snapshot ↔
Checkpoint-Übereinstimmung durch NEUBERECHNUNG:

1. **Hash jeder Seite**: `blake3("SCONE-SNAP-PAGE-HASH" ‖
   vollständige Seiten-Kodierung)` muss dem vom Manifest
   angekündigten Hash gleichen (ein gefälschtes Byte →
   Zurückweisung);
2. **Summe der Einträge == Manifest-Zähler** (eine fehlende Seite
   oder ein falscher Zähler → Zuruckweisung);
3. **SMT-Wurzel neu berechnet aus den importierten Einträgen ==
   `checkpoint.data.state_root`** — DIE kryptografische Prüfung:
   der gesamte Zustand (Domains + TLDs) wird in ein frisches SMT
   neu gefaltet (`snapshot_verify::recompute_state_root`, Blätter
   `SCONE-LEAF-DOM-V1`/`SCONE-LEAF-TLD-V1` exakt wie der lebende
   Pfad), und die Wurzel muss der vom Komitee signierten gleichen.
   Ein gefälschter, abgeschnittener oder unvollständiger Snapshot
   wird VOR JEDEM SCHREIBEN abgelehnt;
4. **strukturelle Konsistenzen**: `height`/`tip_hash`/`state_root`
   des Manifests == die des Checkpoints; strikt steigende Ids
   (keine Duplikate); sequenzielle Seiten-Indizes; DoS-Grenzen
   (≤ 10.000 Einträge/Seite, ≤ 2^20 Seiten, Gesamtbudget).

Verifikations-Fehlschlag → `StorageError::SnapshotRejected`, der
Store bleibt strikt unverändert (alles wird vor der
Schreib-Transaktion verifiziert).

### Serialisiertes Format (strikt, Big-Endian, gnadenloses Dekodieren)

```text
manifest = "SCONE-SNAP-MAN-V1"
        ‖ height(8) ‖ tip_hash(32) ‖ state_root(32)
        ‖ page_count(4) ‖ domain_count(8) ‖ tld_count(8)
        ‖ page_hashes(page_count × 32)

page    = "SCONE-SNAP-PAGE-V1"
        ‖ index(4) ‖ height(8) ‖ state_root(32)      ← jede Seite ist
        ‖ entry_count(4) ‖ entries                   selbst-identifizierend

entry   = kind(0x01 Domain | 0x02 TLD | 0x03 TLD-Untergrenze) ‖ id(32)
          ‖ [len(2) ‖ state | floor(8 BE)]
          (state = DomainStateBytes 57/89 B oder TldStateBytes 2/34 B —
           die bestehenden kanonischen Speicherformate, wiederverwendet; kind
           0x03 (Lebenszeit-Untergrenze einer offenen TLD, Owner-Option) hat
           ein dediziertes Format OHNE Längenpräfix:
           id(32) ‖ floor(8 BE), feste Größe)
```

Seiten-Reihenfolge: alle Domain-Einträge (steigende Ids), dann
alle TLD-Einträge (steigende Ids), dann die auf der
Snapshot-Höhe EINGEFRORENEN TLD-Lebenszeit-Untergrenzen
(`snapshot_tld_floors_all` — nicht die lebende Tabelle) — die
Reihenfolge der Cursor-Paginierung des Stores. Jede Seite trägt
`(index, height, state_root des Checkpoints, entries)`: eine
einzelne Seite weiß, zu welchem Snapshot sie gehört. Seiten von
256 Einträgen beim Export (`VERIFIED_SNAPSHOT_PAGE_ENTRIES`);
DoS-Grenzen beim Import. TLD-Untergrenzen (`tld_floors_v1`) sind
Teil des portablen Formats: kind 0x03 trägt sie, und der Import
verlangt sie strikt steigend und auf eine im Snapshot VORHANDENE
TLD zielend — sonst würde der Importeur nie töten, was ein
Exporteur töten würde (stiller Fork, abgelehnt).

### Export (`export_verified_snapshot(store, checkpoint)`)

Serialisiert den Zustand des LETZTEN FINALISIERTEN CHECKPOINTS —
nicht die Spitze. Emittiert Domains, TLDs, dann die auf der
Snapshot-Höhe eingefrorenen TLD-Lebenszeit-Untergrenzen (Export/
Import-Symmetrie: der Export ließ historisch die Untergrenzen aus
— behoben, Regressionstest
`export_carries_tld_lifetime_floors`). Voraussetzung: der Store
hält einen M6b-persistierten Snapshot (`snapshot_v1_*`) exakt auf
der Höhe des Checkpoints (das `CHECKPOINT_KEEP`/`SNAPSHOT_INTERVAL`-Fenster
garantiert, dass ein solcher Zustand regelmäßig verfügbar ist);
der Anker des Blocks wird aus den gespeicherten Bytes NEU
BERECHNET und muss `checkpoint.data.block_hash` gleichen. Sonst →
`StorageError::SnapshotUnavailable` (Export ist eine explizite
Aktion: auf das nächste Intervall/der nächsten Checkpoint warten).

### Import (`import_verified_snapshot(store, checkpoint, manifest, pages)`)

Verifiziert alles (siehe oben), dann schreibt er in EINER redb
Transaktion: lebende Tabellen `domains`/`tlds` + Zähler + `tip =
(H, Checkpoint-Hash)`, eingefrorene Tabellen `snapshot_v1_*`,
verankert auf demselben Block, und den reservierten Marker
`meta["verified_snapshot_v1"]` (`height ‖ tip_hash ‖
manifest_hash`). Ziel: nur leerer Store (sonst
`SnapshotUnavailable`).

### Verifikationskosten (dokumentiert)

SMT-Rekonstruktion über geordnete Einfügungen: N Einträge × O(40)
Hashes ≈ O(N·40). ~2,1 M blake3-Hashes für 100.000 Domains
(Dutzende ms). EINMALIG — ein Bootstrap-/Reparatur-Import, nie ein
Pfad pro Block: akzeptabel. Geordnete Einfügung (steigende Ids)
erschafft jeden Verzweigungspunkt nur einmal; in jedem Fall hängt
die Wurzel nur vom Inhalt ab (SMT-Garantie, differentieller Fuzz
in `smt.rs`).

## Zustands-Bootstrap — ohne Genesis-Replay (P0.4)

Ein neuer Knoten tritt dem Netzwerk bei, ohne die Historie ab
Genesis abzuspielen:

```text
finalisierter Checkpoint (Relay, von der Chain-Schicht validiert)
        │
        ▼
verifizierter Snapshot (Manifest + Seiten, kryptografische Verifikation oben)
        │  Import → Zustand bei H + Spitze (H, Hash) + Marker
        ▼
Ankerblock H (vom Netzwerk NACH HASH angefordert — append_bootstrap_anchor)
        │
        ▼
jüngste Blöcke H+1..H+k (Relay, steigende Höhe, gewöhnliches atomares
        │               Anhängen — die Speicherungs-API verlangt nur das)
        ▼
synchronisiert — der Boot spielt NUR das Suffix über H ab
```

- **Speicherungs-API dem Relay gegenüber exponiert**:
  `append_bootstrap_anchor(H, hash, bytes)` (legt den Anker-Block
  ab: erforderlicher Marker, Höhen-/Hash-Konsistenz mit dem
  Marker, idempotent), dann gewöhnliches atomares Anhängen nach
  steigender Höhe. Der Netzwerk-Abruf (nach Hash aus dem Manifest)
  ist eine Relay- nicht eine Store-Angelegenheit.
- **`load_chain`** erkennt den Marker: die Historie unter H ist
  legitim abwesend (Nicht-Archiv-Knoten). Konkret: der
  M8-Anti-Replay-Fenster-Rescan ist bei H unterboden (Blöcke unter
  H existieren nicht und fehlen nicht); eine Spitze ohne
  gespeicherten Block NACH einem Import (Anker noch nicht gelegt)
  → typisierte Diagnose `SnapshotUnavailable` („der Anker muss aus
  dem Netzwerk kommen“), keine Korruption.
- **Archiv- vs. Vollzustands-Knoten**: ein Archivknoten
  (Betreiber-Flag) behält alles — er bedient historische Blöcke dem
  Netzwerk und bootet über die bestehenden Pfade. Ein
  Vollzustands-Knoten (P0.4-Standard für einen so KONFIGURIERTEN
  neuen Knoten) bootet bei H: er validiert zukünftige Blöcke
  (Suffix-Replay), kann aber die Historie unter H nicht bedienen
  oder die Genesis nicht abspielen. **Das Standardverhalten ändert
  sich NICHT**: dieser Pfad ist nur aktiv, wenn der Betreiber den
  Import explizit auslöst (`import_verified_snapshot` ist eine
  dedizierte Aktion, nie automatisch); jeder bestehende Knoten
  bootet identisch.
- **Gleichheit mit dem vollen Knoten**: nach dem Anschluss des
  Suffixes H+1..H+k sind Zustand, Spitze und `state_root_smt`
  bit-exakt gleich einem vollen Knoten auf derselben Spitze
  (getestet); der Anti-Replay-Index deckt exakt die Txs auf/über H
  ab — ein Nicht-Archiv-Knoten kann das Fenster unter H nicht
  kennen, die Zustandsregeln bleiben die Absicherung (eine
  duplizierte Tx unter H scheitert am Zustand, wird nie doppelt
  akzeptiert).

## Speicher-Richtlinie

- **Überall Cursor**: `iterate_domains` begrenzt die Seite auf
  `max` Einträge (standardmäßig 100) über redbs träge `range`; die
  gesamte Domain-Tabelle wird nie materialisiert. Große Listen
  (TLDs, Domains) laufen zwingend über diesen Cursor
  (`iterate_tlds`, derselbe Vertrag, M7d).
- **Nur Deltas**: `append_block_with_state` schreibt nur die vom
  Block modifizierten Domains und TLDs — nie den vollen Zustand;
- **Keine synchrone Duplikation**: der RAM-Zustand wird nicht pro
  Block neu persistiert; der Festplatten-Zustand wird nicht pro
  Block neu geladen;
- **Punkt-Lesungen**: jede Lesevorgang öffnet eine kurze
  Lese-Transaktion, gibt den Wert zurück, schließt; redb
  `AccessGuard`s leben nur für die Dauer der begrenzten Kopie;
- **Historische Blöcke nie im RAM**: `load_chain` lädt nur die
  Spitze; `load_chain_replay` streamt Höhe für Höhe;
- **Gepflegter Zähler**: `domain_count` ist O(1), kein Scan.

## DoS-Grenzen

Der Store begrenzt, was ein feindseliger Peer ihn persistieren
oder allozieren lassen kann:

- **`MAX_DHT_CACHE_ENTRY` (64 KiB)**: ein größerer DHT-Eintrag
  wird vor jeder Schreib-Transaktion durch `put_dht_cache`
  abgelehnt → `StorageError::TooLarge` (nichts wird geschrieben);
- **`MAX_DOMAIN_PAGE` (10.000)**: `iterate_domains` begrenzt
  *intern* die angeforderte Seite auf `min(max, 10_000)` — ein
  überhöhtes `max` materialisiert nie mehr als 10.000 Zustände im
  RAM; `iterate_tlds` hat dieselbe Grenze (`MAX_TLD_PAGE`, M7d);
- **Reservierte `meta`-Schlüssel**: `meta_set` lehnt `tip`,
  `tip_height`, `format_version`, `domain_count`, `tld_count` und
  `verified_snapshot_v1` (P0.3) mit
  `StorageError::ReservedKey` ab — das Überschreiben der internen
  Buchhaltung des Stores würde ihn still korrumpieren.
- **Doppelt begrenzter DHT-Cache (LRU)**:
  `DHT_CACHE_MAX_ENTRIES` (100.000 Einträge) UND
  `DHT_CACHE_MAX_BYTES` (2 GiB — Wächter für den Feindschaftsfall;
  jeder Eintrag ist bereits durch `MAX_DHT_CACHE_ENTRY` auf 64 KiB
  begrenzt), welche Grenze zuerst trifft, verdrängt nach
  Schlüssel-Reihenfolge. Die Zähler (Einträge + Bytes) werden zur
  Schreibzeit gepflegt — nie ein Scan zum Zählen; der exakte
  Neuaufbau geschieht nur beim Öffnen (der einzige volle Scan,
  beim Boot). Zugriffsfunktionen
  `dht_cache_entries`/`dht_cache_bytes`, regelbare Grenzen über
  `set_dht_cache_bounds` (Test:
  `dht_cache_respects_the_double_bound`).

## Vertrauensmodell

Zwei Ladepfade, zwei Garantiestufen:

- **`load_chain` — lokales Festplatten-Vertrauen**: der Hash der
  Spitze wird aus dem Header des Spitz-Blocks **neu berechnet** und
  mit dem gespeicherten `meta["tip"]` verglichen (Abweichung →
  `StorageError::Corrupted`, nie eine auf einem gefälschten Hash
  wiederhergestellte Chain), und die Domain-Zustände werden strikt
  dekodiert. Historische Blöcke werden jedoch **nicht abgespielt**:
  die persistierten Zustände werden für bare Münze genommen. Das
  ist der schnelle Start eines Knotens, der seiner eigenen
  Festplatte vertraut (nur von demselben Knoten geschrieben,
  atomar). Seit M6b nutzt dieser Pfad zuerst den
  **Zustands-Snapshot**, wenn benutzbar (siehe unten): der Anker
  des Snapshots wird neu berechnet, und das Suffix über H wird wie
  lebende Blöcke revalidiert — der abgespielte Teil ist somit
  Null-Vertrauen, die eingefrorene Basis Festplatten-Vertrauen.
- **`load_chain_replay` — Null-Vertrauen**: alle Blöcke werden
  einzeln neu gelesen und revalidiert (vollständige Validierung,
  identisch lebenden Blöcken); die resultierende Spitze muss der
  gespeicherten gleichen. Das ist das Kontroll-/Reparatur-Werkzeug.

**Empfehlung**: ein strenger Knoten (oder ein Knoten, dessen
Speicherdatei manipuliert worden sein könnte — geteilte
Festplatte, Wiederherstellung eines dubiosen Backups) muss über
`load_chain_replay` starten, nicht über `load_chain`.

## Nebenläufigkeit

redb 4.x erlaubt **ein einziges `Database`-Objekt pro Datei und
pro Prozess** (gleichzeitiges doppeltes `open` scheitert mit
`DatabaseAlreadyOpen`). Das unterstützte Modell: ein geklonter
`RedbStore` (`Arc<Database>`), über Threads geteilt — Schreiber
serialisiert durch `&mut self` + redbs interne Sperre, gleichzeitige
Leser (Snapshot-Lese-Transaktionen). Nachdem alle Handles
vollständig fallen gelassen wurden, kann die Datei wieder geöffnet
werden.

## Tests (`tests/redb_store.rs` + `#[cfg(test)]`-Module)

Jeder Test verwendet seine eigene redb-Tmpfile (`tempfile`).
Abdeckung:

- Block-Roundtrip Genesis→N, Neustart, Neu-Lesen, Replay;
- M7d: TLD-Register durch atomares Delta persistiert, beim
  Neustart wiederhergestellt (eine `RegisterDomain` unter einer vor
  dem Neustart beanspruchten TLD bleibt zulässig — Fenster
  geschlossen), exakte Zähler beim Wieder-Öffnen, aufsteigender
  `tlds`-Cursor, gescheitertes Anhängen hinterlässt keine TLD,
  reservierter Schlüssel `tld_count` abgelehnt;
- Atomarität: gescheitertes Anhängen hinterlässt nichts (Spitze,
  Domains, Index intakt); doppelte Deltas einmal gezählt;
- Domain-Cursor: Paginierung 100/7, aufsteigende
  `DomainId`-Reihenfolge, Start strikt nach einer existierenden
  oder fehlenden Id, `max = 0`;
- exakter Zähler `domain_count` (Einfügen/Aktualisieren/Neu-Öffnen);
- DHT-Cache-Roundtrip + Überschreiben;
- Nebenläufigkeit: Lesen aus zwei Threads auf geklonten Handles;
- korrumpierte Datenbank: abgeschnittene / Müll-Datei →
  typisierter Fehler, keine Panik (`catch_unwind`);
- M6b: Snapshot im Intervall geschrieben, strikt unter der Spitze;
  Boot aus Snapshot = vollständiges bit-exaktes Replay (state_root
  + SMT, vollständiger logischer Zustand, Spitze); veränderter
  Snapshot (korrumpierter tip_hash, nicht dekodierbarer Zustand,
  Höhe > Spitze, Meta falscher Länge) → stiller Fallback auf
  volles Laden, dieselbe finale Chain; abgespieltes Suffix <
  Intervall; Snapshot-Ersetzung (keine Akkumulation) beim
  Überschreiten des Intervalls; vor-M6b-Store ohne Snapshot-Tabellen
  lädt identisch;
- M8: Anti-Replay-Index beim Boot neu gebaut — eine Tx, die
  innerhalb des Fensters wiedereingebunden wird, wird nach
  Wiederherstellung abgelehnt (`TxReplay`); eine Tx älter als das
  Fenster wird zugelassen (Beschneidung unverändert,
  Zustandsregeln entscheiden); exakte Anzahl == lebender Index (==
  `load_chain_replay`) für einen Store kürzer und länger als das
  Fenster; Snapshot-Boot: der Teil des Fensters unter H wird durch
  den Rescan neu gebaut, der Teil darüber durch das Suffix-Replay;
- P0.2: Namensindex — `resolve_name == DomainId::from_name` für
  120 Domains (lebend, nach Neustart, nach Löschen + Neuaufbau),
  `resolve_tld` für die beanspruchte TLD; GC einer abgelaufenen
  Domain entfernt den Eintrag (keine Geister), vor und nach dem
  Neuaufbau; strenge Trennung der Domain/TLD-Namespaces; ein
  gescheitertes Anhängen hinterlässt keinen Index-Eintrag; der
  Index-Schlüssel ist die dokumentierte BLAKE3-Ableitung;
- P0.3/P0.4 (`tests/verified_snapshot.rs`): Export → Import →
  Boot == vollständiges Replay (Zustand, Spitze, bit-exakte
  `state_root_smt`; der importierte Zustand ist der des
  finalisierten Blocks, nicht die Spitze); Seite durch ein Byte
  gefälscht → Zurückweisung vor jedem Schreiben (Store strikt
  unverändert); fehlende Seite → Zurückweisung; falsche
  `state_root` (Manifest ↔ Checkpoint, dann neu berechnete Wurzel
  ≠ signierte Wurzel) → Zurückweisung; falsche Zähler / Höhen /
  inkonsistente Spitzen / doppelte Ids → Zurückweisung; Import
  verlangt einen leeren Store; Export verlangt einen persistierten
  Snapshot auf der Höhe des Checkpoints; Anker-Vertrag
  (`append_bootstrap_anchor`: erforderlicher Marker, Konsistenz,
  Idempotenz); strenges serialisiertes Format (überschüssige Bytes,
  Tags, DoS-Grenzen); Snapshot bei H dann Blöcke H+1..H+k
  angeschlossen → Chain identisch dem vollen Knoten
  (Zustand/Spitze/Wurzel), Boot ohne Historie unter H,
  Anti-Replay-Index = exakt die Txs auf/über H.

[`ChainState`]: ../../../crates/scone-blockchain/src/state.rs

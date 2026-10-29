# Sicherheitsmodell — Komiteeauswahl und Anti-Grinding

> Normatives Dokument. Jede Abweichung zwischen Dokumentation und
> Code = Bug.

## Überblick

Das Komitee der Epoche N+1 wird aus dem Samen
`next_seed(signing_hash(checkpoint N)) = blake3("SCONE-SEED-V1" || signing_hash(checkpoint N))`
gezogen, über den **Berechtigungspool, der beim von Checkpoint N
finalisierten Block eingefroren wurde**. `signing_hash` ist der
Hash des **Inhalts** des Checkpoints (Epoche, Höhe, Block, prev,
State-Root — Signaturmenge ausgeschlossen): der Samen ist eine reine
Funktion von WAS finalisiert wurde, nie davon, WELCHE
Quorum-Teilmenge es signiert hat — zwei Sichten mit verschiedenen
(beide gültigen) Teilmengen desselben Checkpoints wählen
dasselbe Komitee.

## Bedrohung: Grinding des Komitee-Samens

Ein Angreifer will die Zusammensetzung des Komitees beeinflussen.
Drei Angriffsfenster existieren; alle drei sind strukturell
geschlossen:

### 1. Vor dem Samen — unmöglich, die eigene Position zu grinden

Der Samen der Epoche N+1 leitet sich vom **Hash des Checkpoints N**
ab, quorum-signiert (floor(2n/3)+1 des Komitees N — `quorum_for`).
Checkpoint N committet die `state_root` eines Zustands, der den
Pool **fixiert** (PoS-Basis = exakter Zustand beim finalisierten
Block, `FinalizedBase.pool`). Einen anderen Samen zu produzieren
würde einen anderen quorum-signierten Checkpunkt erfordern — das
heißt, ≥ 1/3 des Komitees N zu korrumpieren, was die vom
BFT-Modell ausgeschlossene Fehlerhypothese ist.

### 2. Nach dem Samen — zu spät zum Beitritt

Jede Registrierung (Domain/TLD-Claim, Erscheinen eines neuen
Owner-Schlüssels) zählt nur für nachfolgende Epochen, deren Samen
noch nicht bekannt sind. Der Pool ist in `FinalizedBase`
eingefroren, BEVOR irgendjemand (einschließlich der Signer des
Checkpoints N) den Samen von N+1 berechnen kann.

### 3. „Best-of-m“-Auswahl ohne Vorteil

Die Position eines Schlüssels in der Ziehung =
`blake3("SCONE-COMMITTEE-V1" || seed || pk)` (uniform), dessen
einziger Freiheitsgrad `pk` ist. Ein Grinder, der m Schlüssel
erzeugt und nur den am besten positionierten registriert, muss
zuerst den **Samen kennen** — aber zu diesem Zeitpunkt ist der Pool
bereits eingefroren. M Kandidaten „für alle Fälle“ zu
registrieren kostet m Proofs of Work (eins pro Domain/TLD) und
ergibt genau m/|pool| erwartete Präsenz: **kein Vorteil gegenüber m
direkten Registrierungen**. Die adaptive Strategie (den Samen
sehen, dann wählen, was man registriert) ist zeitlich unmöglich.

### 4. Letzter-Produzent-Verzerrung (Zurückhaltungsangriff)

Der Produzent des letzten Blocks vor einem Checkpoint könnte
seinen Block zurückhalten, um `signing_hash(checkpoint N)` und
damit den Samen zu beeinflussen. Zwei Schutzmaßnahmen:

- der Checkpoint signiert einen Zustand auf Höhe H, und der **Pool
  wird zum Zeitstempel des Blocks H ausgewertet** — einen Block
  zurückzuhalten verzögert den Checkpoint, ändert aber nicht den
  Pool einer Epoche, die bereits von Checkpoint N−1 eingefroren
  wurde (Pools überlappen wenig von Epoche zu Epoche);
- das Zurückhalten kostet dem Angreifer seine eigene
  Produktionsbelohnung und betrifft nur LIVENESS (verzögerter
  Checkpoint), nie SAFETY: zwei inkompatible finale Checkpoints
  würden ≥ 1/3 Doppelsignaturen erfordern, durch `SlashTx`
  beweisbar und mit Bannen bestraft.

Schlussfolgerung: ein VRF oder Commit-Reveal würde hier nichts
hinzufügen — der Samen ist bereits Funktion eines
quorum-signierten Ereignisses, das unvorhersehbar-verzögerbar-aber-unfälschbar
ist, und das Einfrieren des Pools entfernt den einzigen adaptiven
Hebel. Das Protokoll nicht verkomplizieren (Einschränkung P1.7:
„falls nötig“ — Analyse: nicht nötig).

## Äquivokation und Slashing

Ein Anker, der zwei widersprüchliche Checkpoints in derselben
Epoche signiert, kann von jedem bewiesen werden, der beide
Signaturen besitzt (`SlashTx`, Diskriminant `0x15`): Verbannung aus
dem PoS-Pool (journliert, bei Reorg rückgängig). Die
Domains/TLDs des Beschuldigten werden nicht eingezogen (minimale
Sanktion — M9-Entscheidung).

## Passiver Erwerb und PoS-Berechtigung

**Frage**: kann ein Dritter mir einen Besitz aufzwingen — und damit
eine PoS-Verpflichtung — ohne meine Zustimmung?

**Ja zum Besitz, nein zur Schuld.** Ein Name kann passiv erworben
werden: Abtretungstransaktionen (`TransferTld`, `TransferDomain`)
werden vom **Übertragenden** allein signiert (der Empfänger ist nur
eine opake `OwnerId`, nie konsultiert), `AssignDomain` wird vom
TLD-Eigentümer signiert, und die Genesis-Allokationen landen auf dem
Schlüssel des Manifests. Das ist ein abgesegnetes Feature: Geschenke,
Dead Drops, Vermächtnisse. Die Anti-Sybil-Eigenschaft hängt nicht
daran: ein Sitz pro **unterschiedlichem Owner eines lebenden Namens**
(`owner_refcounts`), und jeder Besitz kostet jemanden Arbeit.

**Keine auferlegbare Schuld**: Slashing bestraft nur **bewiesene
Äquivokation** (zwei widersprüchliche Signaturen, `SlashTx`, Bann);
ein passiver Schlüssel signiert nichts und kann daher nie geslasht
werden. Das Fehlen eines gezogenen Mitglieds geht an **Recovery**
(Neuziehung), ohne Sanktion. Ein unbeabsichtigt erhaltener Sitz
bleibt abgebbar: Ablauf + GC, `SetTldOpen` (der Sitz wird bei der
Öffnung freigegeben), `RevokeTld` oder Re-Transfer.

**Dokumentierte Entscheidung — keine Erwerbs-Zustimmung**: ein
zweiphasiger Claim (ausstehender Transfer, vom Empfänger signierte
Annahme) wurde erwogen und abgelehnt — er bricht Dead Drops und
„first applied wins“, fügt eine Zustandsmaschine über den Namen hin
und schützt vor einem demonstrativ nicht existierenden Risiko (keine
auferlegbare Schuld, siehe oben).

## Durch Tests verifizierte Eigenschaften

- `checkpoint_hash_engages_signatures`: der Checkpoint-Hash deckt
  die Signaturen ab (keine nachträgliche Manipulation);
- `seed_chain_and_recovery`: die Samenkette leitet sich exakt ab,
  Recovery k eingeschlossen;
- `small_committee_never_finalizes`: BFT-Wächter — ein Komitee
  unter `MIN_FINALITY_COMMITTEE_SIZE` finalisiert nie;
- e2e 4 Relays: ein Checkpoint finalisiert und verbreitet sich,
  dieselbe Epoche auf allen Knoten.

## Replay-Angriffe auf die delegierte Autorisierung (Zähler-Bindings)

Fünf Transaktionen mutieren, WER auf einem Namespace agieren darf:
`TransferTld`, `TransferDomain` (Eigentum), `SetRecordDelegates`
(Record-Delegate-Schlüssel), `SetTldOperator` / `SetDomainOperator`
(Betriebs-Delegates). Eine aufgezeichnete Transaktion dieser
Varianten bleibt kryptographisch FÜR IMMER gültig (die Signatur
deckt statische Felder; der Abtretungs-PoW löst einen aus dem
Namespace selbst abgeleiteten Challenge). Was altert, ist nur das
begrenzte Replay-Fenster (letzte 256 Blöcke). Jenseits davon,
Re-Arm-Analyse:

- **Abtretung A→B→A**: A überträgt an B, B zurück an A — der
  Zustand kehrte zu einer früheren Konfiguration zurück. Das
  Wiederabspielen der ORIGINALEN A→B-Transaktion führt die
  Übertragung ohne Zustimmung erneut aus (der Owner-Check geht
  wieder durch; ein statischer Challenge-Beweis löst weiterhin).
- **Delegate-Set-Zyklus**: Delegates gesetzt auf `{9,10}`, geleert
  zu `∅`, dann JEDE Transaktion, die `{9,10}` wiederherstellt — das
  Replay des ersten `SetRecordDelegates` belebt widerrufene
  Schlüssel wieder.

Ein als reine Funktion des AKTUELLEN Zustands berechnetes Binding
(Owner, Set-Inhalt, ...) kehrt zu seinem alten Wert zurück, genau
wenn der Zustand legitimerweise dorthin zurückkehrt — es würde das
Replay wieder scharf schalten, das es töten muss. Die Verteidigung
ist eine MONOTONE Generation: jeder Namespace trägt `mutations: u64`,
von jeder der fünf geschützten Transaktionen inkrementiert; eine
Transaktion bindet sich an den Zähler der Generation, für die sie
gebaut wurde (`BLAKE3-256("SCONE-BINDING-MUTATIONS-V1" ‖ id ‖
generation ‖ counter)`), und die Anwendung verifiziert dieses
Binding gegen den AKTUELLEN Zähler. Zähler kehren nie zurück; daher
armt sich kein Replay wieder. Der Generationsmarker (`registered_at`
für Domains, `valid_until` für TLDs) hält frühere LEBEN eines
neu registrierten Namespace disjunkt.

Residualkante (dokumentiert): das Binding hängt nicht vom
SCHLÜSSEL des Unterzeichners ab. Alle fünf geschützten
Transaktionen sind vom AKTUELLEN Owner des Namespace signiert, also
erfordert ein Inter-Generationen-Replay zusätzlich, dass derselbe
Owner den Namespace in beiden Generationen hält und der Zähler den
aufgezeichneten Wert wieder erreicht — ein selbst kontrolliertes
Szenario für Delegates/Operatoren, und ein Owner-Opfer-Fall für
Abtretungen, der verlangt, dass das Opfer seinen eigenen toten
Namespace Jahre später neu geclaimt hat. Vor dem Mainnet
akzeptiert; neu bewerten, falls Transfervolumen oder
Namespace-Recycling es materialisieren.

## Bekannte Grenzen (dokumentiert, unbehandelt)

- Der berechtigte Pool umfasst Eigentümer lebender TLDs (Claim =
  aufgewendetes PoW) — Sybil-Oberfläche begrenzt durch die
  PoW-Kosten pro Schlüssel;
- die `state_root` des Checkpoints wird nur im RAM verifiziert,
  wenn der Checkpoint die Spitze anzielt; retroaktives Replay ist
  ein Speicherungs-/Relay-Pfad (siehe
  `docs/technical/blockchain.md`).

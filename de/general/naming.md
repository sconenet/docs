# Namensgebung

## Alphabet

Kleinbuchstaben ASCII, Ziffern und Bindestriche (LDH, RFC 1123):
`[a-z0-9-]`, wobei der Bindestrich weder an erster noch letzter
Stelle eines Labels stehen darf. Keine Großbuchstaben, kein
Unterstrich.

Die Validierung ist für ASCII **strikt**: ein nicht-kanonischer Name
wird abgelehnt, nie korrigiert. `"ABC"` → Fehler (kein stilles
Kleinmachen). Begründung: eine Protokollgrenze muss deterministisch
sein; implizite Kanonisierung erzeugt Identitätsmehrdeutigkeiten.

## Internationalisierte Namen (IDNA)

Nicht-lateinische Namen werden als **Eingabe** akzeptiert und zu
Punycode kodiert (IDNA, RFC 5890/5891, UTS#46): `例子.uip` hat
`xn--fsqu00a.uip` als kanonische Form.

- die kanonische Form (on-chain, BLAKE3-Ids, DNS, Speicherung) ist
  immer das ASCII-**A-Label**: jede LDH-Regel oben gilt für diese
  Form;
- die ursprüngliche Unicode-Form (**U-Label**) bleibt zur Anzeige im
  Speicher (`DomainName::unicode()` / `TldName::unicode()`);
- die 63-Byte-Grenze pro Label und die 253-Byte-Gesamtgrenze gelten
  für die kanonische A-Label-Form, nicht für rohes UTF-8;
- ASCII bleibt auf dem strengen Pfad: IDNA gilt nur für
  Nicht-ASCII-Eingaben (`"ABC"` bleibt ungültig);
- Anti-Homograph-Politik (gemischte Schriften, Verwechslungszeichen):
  vorerst nicht gefiltert, nur UTS#46.

## TLD

Eine TLD ist ein DNS-Label wie jedes andere: die generische
Label-Grenze (RFC 1035: 63 Bytes) gilt, keine kürzere TLD-spezifische
Grenze.

- Muster: `[a-z0-9-]{1,63}` (LDH, kein führender/abschließender
  Bindestrich), auf der kanonischen A-Label-Form
- kodierte Grenze: `TldName::MAX_LEN = 63` Bytes
- gültig: `uip`, `com`, `test`, `abc12`, `0x`, `a-b`, eine TLD mit
  genau 63 Bytes, `éxemple` (→ `xn--exemple-2ua`), `例子`
  (→ `xn--fsqu00a`)
- ungültig: eine TLD, deren A-Label 63 Bytes überschreitet, `ABC`,
  `-ab`, `ab-`, leere Zeichenkette

## Domains

- Form: `label[.label]*.tld` — mindestens zwei Labels (`name.tld`);
- jedes Label: LDH `[a-z0-9-]{1,63}`, Bindestrich weder führend noch
  abschließend;
- Gesamtlänge: ≤ 253 Bytes;
- keine leeren Labels (`a..b`, `.a`, `a.` ungültig);
- das letzte Label muss eine gültige TLD sein.

### Subdomains

Der Typ `DomainName` akzeptiert von Anfang an die vollständige
Hierarchie:

```text
example.uip
shop.example.uip
api.shop.example.uip
```

Eine Subdomain hat eine **von der ihres Eltern-Domains
verschiedene** `DomainId`. Regeln zur Subdomain-Vergabe (Rechte des
Eigentümers der übergeordneten Domain usw.) sind zukünftige Arbeit.

## Kanonisierung

Ein validierter Name ist bereits kanonisch: `canonical() == as_str()`.
Deshalb heute abgelehnt: Großbuchstaben, Satzzeichen außer dem `.`
Trennzeichen, Leerzeichen, Unicode, unsichtbare Zeichen usw.

## DomainId

```text
DomainId = BLAKE3-256("SCONE-DOMAIN-V1" || kanonischer_Name)
```

- deterministisch: derselbe kanonische Name ⇒ dieselbe Id, überall,
  für immer;
- praktisch injektiv: zwei verschiedene Namen ⇒ zwei verschiedene Ids;
- der rohe Name darf **nie** als interner Bezeichner dienen
  (Blockchain-Schlüssel, DHT-Schlüssel) — nur für Eingabe und Anzeige;
- das Präfix `SCONE-DOMAIN-V1` isoliert Ableitungsdomänen: eine
  Änderung der Regel (Alphabet, Normalisierung) bedeutet die
  Erzeugung eines neuen versionierten Namespace (ein neues
  Ableitungspräfix), ohne den bestehenden umzuschreiben.

### Textuelle Darstellung

**Kleinbuchstaben**-Hexadezimalkodierung (`0-9a-f`) der 32 rohen
Bytes: genau 64 Zeichen, kein Präfix (`0x`), nie gekürzt.

```text
DomainId([0xab; 32]) → "ababababababababababababababababababababababababababababababababab"
DomainId([0x00; 32]) → "0000000000000000000000000000000000000000000000000000000000000000"
```

Diese Form wird von `Display` erzeugt (das Rust-`Debug` gibt
`DomainId(<hex>)` aus, dieselbe Zeichenkette). Jede Ausgabe, die von
einem Menschen oder einer Maschine außerhalb des kanonischen
Binärformats konsumiert wird — Logs, Fehlermeldungen, CLI-Ausgabe —
MUSS diese Darstellung und keine andere verwenden (keine Kürzung,
kein Präfix, keine Großbuchstaben).

## Grenzen und künftige Regeln (nicht entschieden)

- internationalisierte Namen (IDN) über Punycode;
- Mindestlänge zweiter Labels-Ebenen;
- Ablauf/Verlängerung von Domain-Claims.

## TLDId (M7a)

Eine registrierbare TLD trägt ihre eigene Identität, getrennt von der
jeglicher Domain:

```text
TldId = BLAKE3-256("SCONE-TLD-V1" || tld)
```

- das Präfix `SCONE-TLD-V1` ist **verschieden** von
  `SCONE-DOMAIN-V1`: die Id-Räume von TLD und Domain sind
  konstruktiv disjunkt — die Registrierung der TLD `uip` (ein Claim
  auf den Namespace `*.uip`) kann nie die Identität einer Domain
  besetzen, und umgekehrt;
- dieselben Anzeigeregeln wie `DomainId`: Hex-Kleinbuchstaben,
  64 Zeichen, nie gekürzt;
- angehefteter Vektor (Ableitung per Test angeheftet):
  `TldId("uip") = ad5a86d68643d5c22d6a959bb1a315530c77dfda241f1ac1a8107450f3fab25e`.

Die Transaktion `RegisterTld` (signierter Claim einer TLD, `owner`
immer aus dem eingebetteten Schlüssel neu berechnet) ist seit M7b
Teil der Enum `Transaction` und des Wire-Formats: Diskriminant
`0x93` (beliebige opake Konstante), signierte Nutzlast
`SCONE-TX-SIG-V1`, vollständige Regeln in
`/docs/en/technical/transactions.md`.
Das `RegisterDomain` einer Domain trägt derweil den kanonischen
Namen im Klartext.

## TLD-Reservierung/Governance

Geöffnet mit M7a: `RegisterTld` beansprucht eine freie TLD zugunsten
der aus dem Signaturschlüssel abgeleiteten Identität. Seit M8b ist
die Governance vollständig: Claim mit PoW (schwierigkeit pro Netzwerk), der eine
erneuerbare 3-Jahres-Laufzeit eröffnet (`RenewTld`), Namespace
standardmäßig **geschlossen** (nur assign), explizites Öffnen
(`SetTldOpen`), Übertragung (`TransferTld`), Aufgabe (`RevokeTld`,
neu beanspruchbar). Private TLDs leben 3 Jahre (60-Tage-Karenz,
Verwaltung während der Karenz eingefroren); offene TLDs tragen eine
1-Jahres-Untergrenze, die von ihren Domains erhalten wird; Domains
laufen nach 1 Jahr ab (erneuerbar, 30-Tage-Karenz — siehe
`/docs/en/technical/blockchain.md`).

**ICANN-Root-TLDs**: reserviert. `RegisterTld` auf eine TLD aus der
angehefteten IANA-Liste (`is_icann_tld`, ~1.440 Einträge + RFC 2606)
wird mit einem typisierten Fehler abgelehnt (`IcannTldReserved`) —
der klassische Root gehört dem bestehenden DNS. Zur Auflösungszeit
werden diese Namen an die konfigurierten Upstreams (`--dns-upstream`)
weitergeleitet oder ohne Upstream REFUSED. Von ICANN später
hinzugefügte TLDs bleiben beanspruchbar (angeheftete Liste:
Konsistenz > Aktualität).


## Ökonomie des Lebenszyklus — wer zahlt was, wann

Das Autorisierungsmodell folgt der Natur der Ressource: Arbeit (PoW)
bezahlt, was in die Allmende eingeht; eine Signatur (PoS) verwaltet,
was jemandem gehört. Normative Zusammenfassung:

### Private TLD

- **Geburt — PoW**: per Proof of Work beansprucht, Schwierigkeit nach
  Länge (Mainnet: 1 Zeichen ≈ 34 d Rechenzeit; Untergrenze ≈ 4,3 d ab
  10+ Zeichen). Der Claimant wird Eigentümer und hält einen **PoS-Sitz**
  (Pool der Validator-Berechtigten — Heartbeats zur Frischeerhaltung
  erforderlich, Betreiber-Delegation verfügbar).
- **Leben — PoS**: die Signatur des Eigentümers autorisiert alles im
  Namespace: `assign` (Domain vergeben), `set-tld-open`
  (unumkehrbar — **gibt den Sitz frei**), `transfer`, `revoke` sowie
  die Verlängerung der darunter lebenden Domains.
- **Laufzeit — 3 Jahre**: der Claim eröffnet eine FESTE 3-Jahres-Laufzeit
  (`TLD_TERM_SECS`). **Verlängerung — PoW gleich der Erstellung**
  (`RenewTld`, dieselbe schwierigkeitskurve pro Länge; Challenge
  gebunden an das aktuelle Ablaufdatum — Wiederverwendungs-Schutz):
  jede Lebensspanne wird in Arbeit bezahlt.
- **Ablauf — 60-Tage-Karenz, Verwaltung eingefroren**: nach Fristablauf
  hält eine 60-Tage-Karenz (`TLD_GRACE_SECS`) die Verlängerung offen,
  aber jede Verwaltungsoperation außer der Verlängerung ist
  eingefroren — insbesondere `assign`: ein abgelaufener Eigentümer
  verliert die Verwaltungsrechte, bis er verlängert. Bereits
  zugewiesene Domains darunter leben ihr eigenes Leben (ihr eigenes
  `valid_until`) — die TLD friert ein, nicht ihre Kinder.
- **Tod — GC**: Laufzeit + Karenz ohne Verlängerung → die TLD
  verschwindet (PoS-Sitz freigegeben, von jedermann mit frischem PoW
  neu beanspruchbar). Ein `TransferTld` trägt die Restlaufzeit (kein
  Reset).

### Offene TLD (offen geboren)

- **Geburt — reduziertes PoW** (ein Drittel billiger als privat: eine
  gemeinsame Ressource, die niemand verwaltet). **Kein PoS-Sitz**.
- **Untergrenze — 1 Jahr** (`OPEN_TLD_FLOOR_SECS`): bei der Geburt
  (oder beim Öffnen über `SetTldOpen`) verhindert eine
  1-Jahres-Untergrenze das sofortige Erstellen-Zerstören einer TLD
  ohne Domain.
- **Leben**: niemand signiert etwas — Domains registrieren sich im
  Self-Service. Jede lebende Domain darunter schiebt die
  Todesuhr zurück (sie folgt dem Ablauf der jüngsten Domain,
  begrenzt durch die Untergrenze).
- **Tod — GC**, wenn keine Domain mehr darunter lebt und
  Untergrenze/Ablauf vergangen sind; der Name wird wieder
  beanspruchbar.

### Domain unter einer offenen TLD

- **Geburt — PoW** (reduzierte Kurve ggü. TLD: Mainnet 1 Zeichen ≈
  8,5 d, Untergrenze ≈ ein halber Tag).
- **Verlängerung — PoW identisch mit der Geburt** (gleiche
  Schwierigkeitskurve; die Challenge ist an das aktuelle Ablaufdatum
  gebunden und damit verschieden von der Registrierung und jeder
  früheren Verlängerung): jede Lebensspanne wird in Arbeit bezahlt,
  eine Domain kann nicht gratis „eingesammelt“ werden (≤ 3 Jahre im
  Voraus).
- **Tod**: Ablauf → 30-Tage-Karenz (erneuerbar) → GC → der Name wird
  von **jedermann** mit frischem PoW beanspruchbar.

### Domain unter einer privaten TLD

- **Geburt — das PoS des TLD-Eigentümers**: nur der Eigentümer (oder
  sein delegierter Betreiber) vergibt (`assign`) — die Signatur ist
  die Autorität, kein PoW.
- **Verlängerung — das PoS des TLD-Eigentümers**: vom TLD-Eigentümer
  signiert, leeres `proof`-Feld.
- **Tod**: Karenz dann GC; Neubesetzung läuft erneut über das
  `assign` des Eigentümers.

## Querschnitts-Invarianten

- Jedes PoW wird pro **Epoche** gesalzen, mit einem
  schwierigkeitsproportionalen **rückwärts** gerichteten
  Gültigkeitsfenster (ein mehrwöchiges Mining landet weiterhin, auch
  wenn die Epoche gewechselt hat).
- PoS-Signer-Äquivokation wird **ge-slasht** (lebenslanger Ausschluss
  — M9).
- Jede Operation verifiziert zur Anwendungszeit alles neu: nichts
  wird geglaubt (Hashes, Proofs und Signaturen neu berechnet).

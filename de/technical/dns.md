# UDP- + TCP-DNS-Server (M6 + M7b) — `scone dns`

> Normative Spezifikation der Scone-DNS-Fläche.
> Jede Abweichung zwischen Dokumentation und Code = Bug (die Quelle
> der Wahrheit beheben).

## Überblick

Der **DNS-Server** ist eine optionale Fläche des Relays: er
beantwortet **UDP- und TCP**-DNS-Abfragen (RFC 7766, derselbe
Port) für Scone-Namen **nur aus gegen die Chain verifizierten
Daten** und bietet einen **optionalen rekursiven Fallback**
(`addr:port`) für alles andere. Er besitzt weder Chain noch Store
noch DHT: Existenz-/Ablauf-Entscheidungen laufen über den
**autoritativen DNS-Snapshot** (eine RAM-Sicht, nach jedem
angewandten Block veröffentlicht — siehe unten), und
Eintragsabrufe laufen über das Relay (Prozess), über den internen
`RelayCommand`/`ResolveLocal`-Kanal (In-Process, ein Roundtrip) —
derselbe verifizierte Pfad wie die CLI.

```text
              UDP+TCP :5353 (Standard 127.0.0.1)
                    │
             scone relay --dns 127.0.0.1:5353
                    │  ┌────────────────────────────────┐
                    ├──┤ Shard-Cache, 16 Wege (qname)   │
                    │  └────────────────────────────────┘
                    │  Scone-förmiger Name?
                    │  ├─ JA → autoritativer DNS-Snapshot:
                    │  │        unbekannt/abgelaufen → NXDOMAIN (RAM,
                    │  │        null Roundtrips in den Kern)
                    │  │        → Apex-Suche nach Suffix
                    │  │        → internes `resolve_local` (Relay)
                    │  │           Chain + verifizierter Eintrag (Hash,
                    │  │           Owner, Sequenz) → Antworten
                    │  │        Apex ohne veröffentlichten Eintrag → NODATA
                    │  └─ NEIN → Upstream-Fallback (falls konfiguriert)
                    │           sonst REFUSED
                    │
                    │  Antwort-Budget:
                    │    UDP → 512 Bytes (kein EDNS); Überlauf =
                    │          beim letzten passenden Eintrag geschnitten + TC=1 →
                    │          der Client wiederholt über TCP
                    │    TCP → 64 KiB (2-Byte-Längenpräfix,
                    │          RFC 1035 §4.2.2): volle Menge
```

## Start

```bash
scone relay --dns 127.0.0.1:5353            # nur DNS-Fläche
scone relay --dns 0.0.0.0:53 \
            --dns-upstream 1.1.1.1:53 \
            --dns-upstream 8.8.8.8:53       # + rekursiver Fallback
```

Das Relay kündigt die effektive Adresse auf stdout an
(maschinenlesbarer Vertrag, wie `p2p:`): eine Zeile `dns: <addr>`,
sobald der Socket gebunden ist.

Validierungs-Client: `dig <name> --dns 127.0.0.1:5353
--qtype A` (A, AAAA, TXT, MX, NS, CNAME, ANY).

## Auflösungspfad (AUTORITÄT = die Chain, immer)

0. **Autoritativer DNS-Snapshot**: eine unveränderliche Sicht
   (`Arc<HashMap<DomainId, {valid_until, has_record}>>`,
   `tokio::sync::watch`-Feed), veröffentlicht vom Relay-Kern **nach
   jedem angewandten Block** (Push, nicht Poll) und beim Boot.
   Existenz-/Ablauf-Entscheidungen — NXDOMAIN, abgelaufene Domain —
   werden aus dem RAM über diesen Snapshot bedient: ein `Arc`-Clone
   + ein Map-Nachschlag, **ohne je in die Single-Owner-Run-Schleife
   des Relays einzutreten**. Der volle Pfad
   (`RelayCommand`/`ResolveLocal`) bleibt tatsächlichen
   Eintragsabrufen vorbehalten (DHT-Pulls), dank des Caches selten.
   Der Snapshot trägt NIE Eintragsdaten: nur Existenz und Ablauf.
   - **O(berührt) Copy-on-Write-Deltas**: die Map ist ein
     `Arc<HashMap>`; eine Spitzen-Verlängerung veröffentlicht das
     Delta des Blocks (die eigenen Transaktionen des Blocks nennen
     jede Domain, die sie ändern können — Register/Assign/Update/
     Renew/TransferDomain — plus die zur Push-Zeit ausgewerteten
     Entfernungen der deterministischen Ablauf-GC) über
     `Arc::make_mut`: die Map wird NUR geklont, wenn ein Leser noch
     die vorherige Sicht hält. Boot und Reorgs bauen den
     vollständigen Snapshot neu (ein paginierter Backend-Durchlauf).
   - Frische: ein Leser kann während einer Blockanwendung einen
     wenige Hundert ms alten Snapshot sehen — dieselbe
     Veralterung, die ein Status-RPC hätte; DNS-Cache-TTLs sind
     unberührt (Antworten bleiben autoritativ gemäß ihrer Herkunft).
1. **Shard-Cache**: [`ShardedCache`] — 16 unabhängige `Cache`-
   Shards, verschlüsselt nach `hash(qname)` (qtype teilt den Shard:
   Namens-Lokalität). Ein heißer Name konkurriert mit 1/16 des
   Verkehrs statt hinter einem einzigen Mutex bei Tausenden qps
   anzustehen. GLOBALE Grenzen (standardmäßig 100.000 Einträge /
   64 MiB, `--dns-cache-limit` / `--dns-cache-max-mem`) pro Shard
   geteilt (dokumentierter Worst Case: ein Faktor 1 + N/Shards).
   Die ECHTE rdata-Größe jedes Eintrags wird verrechnet (~128 Bytes
   Overhead pro Eintrag), und die FIFO-Verdrängung löst bei der
   zuerst erreichten der zwei Grenzen aus — die Speicherobergrenze
   hält sogar gegen Einträge in voller Größe
   (`MAX_DHT_RECORD_LEN`). TTL 60 s (autoritativ) / auf 600 s
   geklemmt (Fallback). Der Negativ-Cache (NXDOMAIN/NODATA) folgt
   denselben Regeln; jeder Eintrag trägt seine Herkunft
   (autoritativ oder Fallback), damit das AA-Bit bei einem Treffer
   ehrlich bleibt. **Niemals SERVFAIL cachen** (transienter lokaler
   Fehler) **noch REFUSED** (Konfigurationszustand): beide werden
   bei jeder Abfrage neu berechnet. FIFO-Verdrängung bei Kapazität.
   **Hot Path (Cloudflare/1.1.1.1-inspiriert)**:
   - geteilte `Arc<CacheEntry>`-Einträge: ein Treffer ist ein
     Refcount-Bump, NIE ein tiefes Klonen der rdata (`put` ERSETZT
     das Arc — keine geteilte Mutation; die TTL wird weiterhin unter
     der Shard-Sperre geprüft);
   - **allokierungsfreier Nachschlag**: zweistufige Map
     (`Box<str>` qname → qtype → Eintrag), das geliehene `&str` der
     Abfrage testet direkt (`Borrow<str>`) — der alte
     Tupel-Schlüssel `(String, u16)` allokosierte bei JEDER
     Suche eine frische `String`;
   - **gepackte zusammenhängende Einträge**: ein `Box<[u8]>`-Puffer
     pro Eintrag (`u16 rdlen ‖ u16 rtype ‖ u32 ttl ‖ rdata` pro
     Eintrag) statt eines `Vec` pro Eintrag — das Speicherbudget
     verfolgt exakt, was allokiert ist;
   - **RFC-1035-Namenskompression bei der Ausgabe**: der Owner
     autoritativer Antworten ist ein 2-Byte-Kompressions-Zeiger
     (`0xC00C`) in die zurückgeworfene Frage — die Frage wird nie
     pro Eintrag vollständig neu kodiert.
2. **Scone-förmig?** Strenger Vor-Filter: Labels `[a-z0-9-_]`
   (1..63), ≥ 2 Labels, TLD 1..63 Bytes `[a-z0-9-]` (siehe
   `TldName::MAX_LEN`), Gesamtlänge ≤ 253 — **und TLD außerhalb
   des ICANN-Roots** (`is_icann_tld`, eingefrorene IANA-Liste +
   RFC 2606: `com`, `fr`, `org`… gehören nie zu Scone). Ein Name,
   der KEIN Scone-Name sein KANN (z. B. `www.foo_bar`, Unterstrich
   in der TLD), geht an den Fallback; ein möglicher Scone-Name
   (`absent.uip`) ist eine Chain-Angelegenheit: unregistriert →
   autoritatives NXDOMAIN, selbst ohne Upstream. Eine ICANN-TLD
   (`example.com`) wird, falls konfiguriert, an die Upstreams
   weitergeleitet, sonst REFUSED — nie autoritativ von Scone
   aufgelöst.
3. **Apex-Suche**: Suffixe vom längsten zum kürzesten (min.
   2 Labels). Das erste Suffix, dessen On-Chain-Zustand dem Relay
   bekannt ist = Apex; der Eintragssatz bedient den ganzen qname
   (Subnamen eingeschlossen — implizites Wildcard-Verhalten, offene
   Entscheidung).
4. **`resolve_local` (interner Kanal)**: ein einziger Roundtrip,
   nie ein DHT-Wartender. On-Chain-Zustand (`registered`, Owner,
   Sequenz) + lokaler DHT-Cache-Eintrag **neu verifiziert** (Owner,
   Sequenz, record_hash == On-Chain-Commitment). Ein
   nicht-konformer Eintrag antwortet nie — Verfügbarkeit ≠
   Autorität.
   - unregistriert → NXDOMAIN;
   - registriert ohne gültigen Eintrag → NODATA (NOERROR, 0
     Antworten);
   - verifizierter Eintrag → nach qtype gefilterte Antworten.

## Codec (RFC 1035, strenge Teilmenge)

- Abfrage: 12-Byte-Header, **eine** Frage, QCLASS = IN, **keine
  Kompression** (Zeiger = ablehnen). Labels kleingemacht und
  validiert. QR=1, QDCOUNT≠1, ungültige Labels → **Stille**
  (Müll kostet nichts).
- Antwort: QR|RD(Echo)|RA, **AA nur bei autoritativen Antworten**
  (Chain, frisch oder gecacht — nie beim Fallback noch bei
  Fehler-RCODEs), Frage wörtlich kopiert, autoritative Antworten
  mit einem **RFC-1035-komprimierten Owner** (2-Byte-Zeiger
  `0xC00C` in die zurückgeworfene Frage); Upstream-Antworten des
  Fallbacks werden wie sie sind bedient (nie neu kodiert).
- Bediente Typen: A, AAAA, CNAME, NS, MX, TXT, **PTR, SRV, CAA**
  (+ Unknown wörtlich neu kodiert; insgesamt 9 native Typen, rdata
  kodiert nach RFC — 1035/2782/8659; strenges CAA: nur
  Critical-Bit 0x80, kleingeschriebene Tags). Eine A/AAAA-Abfrage
  schließt die CNAMEs der Menge ein (Alias-Verfolgung). ANY-Abfrage
  (255): die ganze, vom Codec gefilterte Menge.
- Grenzen (M7b): **das Antwort-Budget hängt vom Transport ab** —
  512 Bytes über UDP (klassische RFC-1035-§2.3.4-Nutzlast, kein
  EDNS), 65.535 über TCP. Eine das Budget überschreitende Menge
  wird beim letzten Eintrag, der passt, geschnitten, mit **TC=1**
  (ehrliche Abschneidung, RFC 1035 §4.2.1): der Client wiederholt
  über TCP (RFC 7766), wo die volle Menge bedient wird. Eine
  Upstream-Antwort, größer als das UDP-Budget, wird abgeschnitten
  neu kodiert (TC=1), statt wie sie ist geliefert zu werden; über
  TCP geht sie Byte für Byte durch. Die Abschneidung gilt auch für
  Cache-Treffer (das Budget wird zur Antwort-Zeit angewandt, nicht
  zur Cache-Füll-Zeit).
- RCODEs: 0 NOERROR (mit oder ohne Antworten), 2 SERVFAIL
  (lokaler Fehler), 3 NXDOMAIN (autoritativ), 5 REFUSED
  (nicht-Scone ohne Upstream).

## Rekursiver Fallback

- Upstreams `addr:port` (UDP), der Reihe nach versucht, 2-s-Timeout
  pro Upstream, Socket mit dem Upstream **verbunden** (Kernel-
  Filterung: ein Paket von jeder anderen Quelle wird nie
  akzeptiert). Die Upstream-Antwort wird nur akzeptiert, wenn sie
  **zur Abfrage passt**: dieselbe TXID, QR=1, Frage
  (QNAME+QTYPE+QCLASS) Byte für Byte kopiert. Eine nicht-konforme
  Antwort → SERVFAIL, und nichts wird gecacht.
- Die validierte Upstream-Antwort wird unverändert
  durchgereicht; ihre Antworten werden mit Herkunft `fallback`
  gecacht (AA nie gesetzt, selbst bei einem Cache-Treffer) und die
  TTL auf 600 s geklemmt. Upstream-Fehler-RCODEs (außer
  NOERROR/NXDOMAIN) werden nicht gecacht.
- **Standardmäßig privat**: kein Upstream = REFUSED für Namen
  außerhalb der Scone-Form.

## DHT-Negativcache (Anti-Verstärkung)

Ein **unbeantworteter DHT-Pull** (Eintrag on-chain erwartet —
`record_hash` committet — aber abwesend aus dem lokalen Cache und
den befragten Peers) parkt den Namen für `DHT_NEGATIVE_TTL_SECS`
(30 s) in einem begrenzten Negativ-Cache (FIFO, Cap
`DHT_NEGATIVE_CAP` = 4.096 Namen): wiederholte Auflösungen
desselben Namens innerhalb der TTL werden **unterdrückt**
(gezählt, nie neu gepullt). Ein Scanner — oder ein heißer Name,
dessen Owner nie veröffentlichte — kann nicht mehr eine DNS-Abfrage
in ein `GetRecord` pro Mesh-Peer pro Abfrage verstärken. Der
Negativ-Eintrag stirbt bei TTL, bei Ankunft eines Eintrags (der
erfolgreiche Pull schreibt in den echten Cache) oder durch
Hinausdrängen aus dem FIFO-Cap.

Exponierte Zähler (`scone_dht_negative_hits_total`,
`_expired_total`, `_capped_total` — siehe „Metriken“ in
`relay.md`).

## TCP (M7b, RFC 7766)

Dieselbe Port wie UDP (zweites TCP-Bind auf der Adresse `--dns`).
Der Transport folgt RFC 1035 §4.2.2: jede Nachricht wird mit ihrer
Länge auf **2 Bytes** Big-Endian präfixiert.

- **Abfrage**: Präfix-Länge ≥ 1 und ≤ 512 Bytes (eine DNS-Abfrage
  passt in die klassische Nutzlast — RFC 7766 §6; kein EDNS bedient).
  Außerhalb der Grenzen → sofortiges Schließen der Verbindung.
- **Antwort**: 65.535-Byte-Budget (das 2-Byte-Präfix-Maximum) —
  eine für UDP zu große Menge wird hier **vollständig** bedient,
  TC=0.
- **Pipelining**: mehrere Abfragen pro Verbindung werden seriell,
  der Reihe nach bedient (ein Schreiben = eine präfixierte
  Antwort).
- **Lese-Timeout**: 5 s pro Lesevorgang (Präfix oder Rumpf) — ein
  stiller Client wird getrennt, sein Slot freigegeben.
- **Gleichzeitige Verbindungen**: 64 max (Semaphor); die 65. wird
  sofort abgelehnt (Verbindung geschlossen), nie in Warteschlange —
  kein unbegrenztes Wachstum von Tasks oder Speicher.
- Lese-/Schreib-/Timeout-Fehler: Debug-Log + sauberes Schließen.
  **Nie panisch bei Netzwerk-Eingabe.**
- Müll (unverständliche Abfrage) erhält Stille, schließt aber die
  Verbindung nicht: die nächste (gültige) Abfrage wird bedient.

```text
Client                     Server
  │ [len u16][Abfrage]       │
  │ ─────────────────────►   │ Cache → autoritativ → Fallback
  │ ◄─────────────────────   │ [len u16][Antwort ≤ 64 KiB]
  │ (UDP hatte TC=1 geantwortet → der Client wiederholt hier)
```

## Grenzen und Schutzleitungen

| Schutzleitung | Wert |
|---|---|
| UDP-Datagramm (Abfrage/Antwort) | 4096 Bytes (darüber abgelehnt) |
| UDP-Antwort-Budget (darüber TC=1) | 512 Bytes (kein EDNS) |
| Max. TCP-Abfrage | 512 Bytes (darüber geschlossen) |
| Max. TCP-Antwort | 65.535 Bytes |
| Gleichzeitige TCP-Verbindungen | 64 (Semaphor, sofortige Ablehnung) |
| TCP-Lese-Timeout | 5 s |
| Cache | GLOBALE Grenzen standardmäßig 100.000 Einträge / 64 MiB, über 16 Shards geteilt (`--dns-cache-limit` / `--dns-cache-max-mem`) |
| Autoritative TTL | **pro Eintrag** (`record.ttl`, Grenzen 1..=2.592.000 s; die historische Konstante 60 s bleibt nur für Dienst-Antworten) |
| Fallback-TTL (Klemme) | 600 s |
| Parallele UDP-Abfragen | 4.096 (`MAX_INFLIGHT_UDP`, Semaphor; darüber: verwerfen, der Client wiederholt) — regelbar `--dns-max-inflight` (min 1) |
| DHT-Negativcache | 30 s (`DHT_NEGATIVE_TTL_SECS`), FIFO-Cap 4.096 Namen |
| Timeout pro Upstream | 2 s |

Eingehaltene Projektregeln: nie panisch bei Netzwerkdaten; striktes
Dekodieren überall; der Server berührt weder Chain noch Store noch
DHT (das Relay bleibt die einzige Verifikations-Autorität).

## Tests

- Unit (`scone-network::dns`): strikter Codec (QR, Kompression,
  Label-Zurückweisungen), autoritative Antworten, NODATA, NXDOMAIN,
  REFUSED ohne Upstream, Fallback zu einem UDP-Mock, begrenzter
  Cache, TXT-Stückelung, Upstream-Parsen; M6-Nachbesserung:
  AA-Bit-Ehrlichkeit (autoritativ vs. Fallback vs. Fehler,
  einschließlich Cache-Treffer), TC=1-Abschneidung einer
  übergroßen Menge, Zurückweisung einer Upstream-Antwort (TXID-
  oder Fragen-Abweichung), kein Cachen von SERVFAIL/REFUSED;
  Shard-Cache: Shard-Streuung, geteilte globale Grenzen, 4
  gleichzeitige Worker auf einem heißen Namen; Hot Path:
  allokierungsfreier Nachschlag, gepackte Einträge,
  Owner-Kompression.
- M7b: Budget pro Transport (UDP 512 Bytes → TC=1, TCP → volle
  Menge), Abschneidung eines Cache-Treffers, echtes `run_tcp`
  (Menge > 512 vollständig bedient, Pipelining von 2 Abfragen,
  übergroße Abfrage → Schließen, stiller Client → 5-s-Timeout,
  Müll dann gültige Abfrage auf derselben Verbindung),
  Upstream-Antwort > 512 neu kodiert TC=1 über UDP / Byte für Byte
  über TCP.
- Echte Integration (`tests/dns_server.rs`): ein volles Relay
  (einschließlich Devnet-Produktion) + Registrierung +
  UpdateDomain + PutRecord, dann echte UDP-Abfragen: verifiziertes
  A (AA-Bit), TXT, NODATA AAAA, NXDOMAIN, Fallback über Mock,
  erneute Abfrage, Subname über Apex.
- Binäres E2E (`crates/scone/tests/dns_cli.rs`): die echte
  Binärdatei `scone` — `relay --dns`, `identity generate`,
  `domain register`, `domain update --file`, dann `dig`
  (dnsutils) (verifiziertes A, TXT, NXDOMAIN, REFUSED) auf einem
  echten UDP-Port.

## Offene Entscheidungen

- Kein EDNS (das UDP-Budget bleibt 512 Bytes; TC=1-Abschneidung +
  TCP deckt den Devnet-Bedarf).
- Der Eintragssatz trägt keinen Namen pro Eintrag: die ganze Menge
  bedient jeden Subnamen des Apex (implizite Wildcard). Adressierung
  pro Eintrag würde eine Erweiterung von `DnsRecord` erfordern
  (Protokoll).
- Der Fallback leitet die rohe Abfrage weiter: ein bösartiger
  Upstream kann alles beantworten (unvalidierte Antwort — das ist
  das Standard-Rekursiv-Modell; Scone-Namen gehen NIE durch ihn).

## GeoDNS (geografisches Routing)

> Normative Spezifikation. Jede Abweichung zwischen Dokumentation
> und Code = Bug.

### Eingebettete GeoIP-Tabelle (automatisch)

Jede Binärdatei bettet eine **IP → Land-Zuordnungstabelle** ein,
kompiliert in das Relay: **keine Konfiguration nötig**, das
geografische Routing ist überall automatisch aktiv.

- **Datenquelle**: **DB-IP Lite** („IP to Country Lite“),
  lizenziert unter **CC BY 4.0** — weiterverteilbar mit
  Namensnennung: `https://db-ip.com` (monatlich kuratiert; zuvor:
  die RIR-Delegationen via `scripts/gen-geo-table.py`, entfernt).
  Quantisierte Zellen **/20 v4 und /24 v6** (DB-IPs native
  Granularität, ~1,3 Mio. Bereiche, übersteigt die Speichergrenze
  des Parsers) — ~96.500 Regeln. Das Erzeugungsdatum steht im
  Tabellen-Header (`assets/embedded-geo.table`).
- **Prüfsumme**: der SHA-256 der Tabelle wird zur Kompilierzeit
  berechnet (`build.rs`), **beim Booten erneut verifiziert**
  (Mismatch = typisierte Verweigerung — eine veränderte
  Binärdatei serviert nie geografische Antworten) und beim Booten
  mit der Namensnennung geloggt. Der Hash garantiert, dass „alle
  Knoten eines Builds dieselbe Tabelle teilen“ CHECKBAR ist,
  nicht angenommen.
- **Regenerierung**: `scripts/geo-convert.py` (lädt die
  DB-IP-Lite-CSV des Monats herunter, konvertiert — numerisches
  Land, Kontinent abgeleitet ISO→M49, ASN/Region in der Lite
  nicht vorhanden).
- **Granularität**: Land (numerisch ISO 3166-1) + abgeleitetes
  Kontinent (UN M49). Kein ASN noch Region (in der Lite nicht
  vorhanden — ein ASN/Region-Bedarf wartet auf eine Evolution der
  eingebetteten Pipeline, nie auf eine lokale Tabelle).
- **Leistung**: **Binärsuche** über sortierte Bereiche (Index beim
  ersten Gebrauch einmal aufgebaut, pro v4/v6-Familie) — gemessen
  ~95 ns/Nachschlag heiß, ~50 µs Random-Access auf der vollen
  Tabelle (Bench: `cargo test -p scone-network --release --test
  geo_bench -- --ignored`).
- **Selbst-Lokalisierung des Relays**: das Routing folgt der
  **Quell-IP jeder Abfrage** — ein LAN-Relais leitet seine lokalen
  Clients automatisch in die richtige Zone.
- **AUFERLEGTE Tabelle** (remove-geo-operator-table, 2026-10-22):
  die eingebettete Tabelle ist die **einzige Quelle** — kein
  Operator-Import noch -Override existiert. **Keine Geo-Option zur
  Laufzeit existiert**: jedes `--geo-*`-Flag ist eine harte
  clap-Ablehnung (die Geozone ist vom Build auferlegt). Ein
  Operator mit besonderem Bedarf (ASN, Region) wartet auf eine
  Evolution der eingebetteten Pipeline, nicht auf eine lokale
  Tabelle.

### Überblick

Eine Domain kann **geografische Routing**-Regeln in ihrem
signierten Eintragssatz veröffentlichen: `GEOROUTE`-Einträge
(privater Scone-Typ `32.768`, außerhalb des IANA-Bereichs). Der
Resolver beantwortet A/AAAA-Fragen für diese Domains mit der
**Adresse des von der Regel gewählten Endpunkts** (Scope am
nächsten zur Position des KUNDEN, aus seiner Quell-IP über die
Tabelle abgeleitet), statt eines statischen A/AAAA-Eintrags. Alles
andere (Domains ohne GEOROUTE, andere qtypes) folgt dem
klassischen DNS-Pfad, **Byte für Byte**.

### GeoDNS-Metriken

Der Prometheus-Endpunkt legt die Geo-Antwort-Aufteilung offen:

- `scone_dns_geo_answers_total` — insgesamt bediente
  GEOROUTE-Antworten;
- `scone_dns_geo_by_continent_total{continent="<M49>"}` — nach
  Kontinent;
- `scone_dns_geo_by_country_total{country="<ISO>"}` — nach Land
  (durch die Standards selbst begrenzt: ≤ 9 Kontinente, ≤ 250
  Länder).

### GEOROUTE-Eintrag (Wire)

Ein GEOROUTE = ein **Scope** + ≤ 32 **Endpunkte** (kanonische
Sortierung beim Dekodieren: Priorität aufsteigend, Gewicht
absteigend, Ziel aufsteigend; Duplikat = ablehnen):

```text
GeoScope   = 0x00 (Welt)
           | 0x01 continent.v (UN-M49-Code: AF=2, AS=142, EU=150,
                                NA=3, SA=5, OC=9, AN=999)
           | 0x02 country.v   (numerischer ISO-3166-1-Code: US=840,
                                TW=158, FR=250…)
           | 0x03 region.v    (OPAQUE ISO-3166-2-Unterteilung, u32:
                                vom Veröffentlicher definierte Nummer,
                                wie besehen verglichen — keine lokale
                                Tabelle)
           | 0x04 asn.v       (AS-Nummer, u32)
           | 0x05 prefix.addr[16] || prefix.len u8
             (normalisiertes 16-Byte-IP-Präfix: IPv4 = Adresse in den
              ersten 4 Bytes + NULL-Schwanz, len ≤ 32;
              IPv6 = 16 freie Bytes, len ≤ 128; Host-Bits IMMER
              null — ein nicht-normalisiertes Präfix wird abgelehnt)
GeoTarget  = 0x00 octets[4]  (binäre IPv4-Adresse)
           | 0x01 octets[16] (binäre IPv6-Adresse)
GeoEndpoint = target || priority.v || weight.v
GEOROUTE    = scope || count.v(≤32) || endpoints…
```

Validierungsgrenzen (Dekodieren): `priority > 0`, `weight ≤ 100`
(`weight = 0` = inaktiver Endpunkt, nie gewählt), TTL innerhalb
1..=2.592.000 s wie jeder Eintrag. Ein den lokalen Tabellen
unbekanntes Land/Kontinent ist auf dem Wire **erlaubt**: das
Routing fällt einfach auf einen höheren Scope zurück.

**Wire-Rückwärtskompatibilität (v1 → v2)**: die v1-Bytes sind
UNVERÄNDERT — v1-Scopes (Welt/Kontinent/Land) behalten ihre
Diskriminanten `0x00..=0x02` und ihre exakte Kodierung; die drei
neuen Scopes nehmen die folgenden Diskriminanten (`0x03..=0x05`) in
einer mit der kanonischen Sortierung konsistenten Gesamtordnung
(`World < Continent < Country < Region < Asn < IpPrefix`). Ein
v1-Dekoder, der auf einen Diskriminanten ≥ `0x03` trifft, scheiterte
bereits strikt (Eintrag abgelehnt); v1-produzierte Bytes bleiben
identisch dekodiert (siehe den Test `geo_known_bytes`, eingefroren).

### Auswahl (Auflösung)

1. **Scope** — abnehmende Spezifität, **6 Stufen**: **längstes
   passendes IP-Präfix → exakte AS → exaktes Land → exakte Region
   → Kontinent → Welt**.
   - *IpPrefix*: unter den Routen, deren Präfix zur IP des Kunden
     passt (wie dem Lokalisierer bekannt: `client_ip`, sonst die
     bekannten Ursprungs-Präfixe), gewinnt das **längste** (`len`
     max); bei gleichem `len` das erste in kanonischer Ordnung (die
     Reihenfolge des `routes`-Slice, beim Kodieren sortiert). Eine
     Unterteilung ist weniger spezifisch als ein Präfix/eine AS
     (die das exakte Netzwerk identifizieren) und weniger als das
     Land (opake Nummerierung, nur zuverlässig, wenn Veröffentlicher
     und Lokalisierer dieselbe teilen), aber mehr als das Kontinent
     (sie ist in ihrem Land enthalten, also erst recht in ihrem
     Kontinent).
   - *Asn*: exakte AS-Nummer des Kunden.
   - *Country*: exaktes Land.
   - *Region*: exakte Unterteilung (opaker Bezeichner).
   - *Continent*: vom Lokalisierer bereitgestellt **oder aus dem
     Land abgeleitet** über die lokale Tabelle
     (`continent_of_country`); ein der Tabelle unbekanntes Land
     fällt auf das explizite Kontinent oder die Welt zurück.
   - *World*.
   Ein v1-[`GeoLocation`] (Region/asn/Präfixe/client_ip unset)
   erzeugt exakt das v1-Verhalten: Land → Kontinent → Welt. Keine
   passende Route (einschließlich ohne Welt-Scope) → klassisches
   DNS-Verhalten, als gäbe es den GEOROUTE nicht.
2. **Priorität** — SRV-artige Semantik: nur die Gruppe der
   **minimalen Priorität** ist wählbar; höhere Prioritäten sind
   **Failover**-Stufen.
3. **Gewicht** — innerhalb der gewählten Gruppe **deterministische
   gewichtete** Ziehung: stabiler Hash (SplitMix64) einer Nonce
   (der Abfrage-Zähler des Resolvers) gegen die kumulativen
   Gewichte. Kein RNG: zwei Abfragen mit derselben Nonce geben
   dasselbe Ergebnis, steigende Nonces verteilen proportional zu
   den Gewichten. `weight = 0` = inaktiv, nie gewählt (selbst
   allein — weiter zur nächsten Stufe).
4. **Signierte Reihenfolge, null Prüfungen** — der Resolver
   beurteilt NIE die Verfügbarkeit eines Endpunkts: jeder aktive
   Endpunkt (weight > 0) des gewählten Scopes ist Kandidat, der
   Kopf der minimalen Stufe wird durch die gewichtete Ziehung
   gewählt, und die folgenden dienen als Failover **in der
   Reihenfolge des signierten Eintragssatzes** (die Reihenfolge
   IST der Failover-Mechanismus). Die Erreichbarkeit eines
   Endpunkts ist Sache seines Betreibers, nie die des Resolvers.

### Antwort

- Eine einzige Adresse: **A wenn IPv4, AAAA wenn IPv6** — strenge
  Filterung nach Familie gemäß dem qtype (eine A-Frage antwortet
  NIE mit einem IPv6, und umgekehrt).
- TTL = die des Quell-GEOROUTE-Eintrags (beim Dekodieren durch die
  allgemeinen TTL-Regeln begrenzt).
- AA gesetzt (autoritative Antwort — der GEOROUTE ist ein
  verifizierter signierter Eintrag wie jeder andere).
- Leere Antwort nach Filterung (kein Kandidat der richtigen
  Familie, keine Route für die Zone) → der Resolver
  fährt auf seinem klassischen Pfad fort (typischerweise NODATA:
  NOERROR, 0 Antworten); Fehler-RCODEs folgen den allgemeinen
  Regeln (nie gecacht).
- Ein klassischer A/AAAA-Eintrag, der im Satz koexistiert, hat
  VORRANG vor Geo (Geo gilt nur bei Abwesenheit einer
  Wire-Antwort für den qtype).

### Cache: explizite Entscheidung

Die Geo-Routing-Entscheidung wird **NIE** unter `(qname, qtype)`
gecacht: sie hängt vom Kundenscope des Resolvers ab, und ein
solcher Schlüssel würde die US-Route einem TW-Kunden servieren.
Geo-Antworten werden bei jeder Abfrage neu berechnet (Kosten: eine
select_scope + gewichtete Ziehung, µs) und umgehen den Cache. Der
klassische (qname, qtype)-Cache bleibt für alles andere unverändert —
einschließlich des NODATA, das entsteht, wenn Geo nichts entscheidet.
Fehler-RCODEs bleiben nie gecacht.

### NS-Delegierung (`ns`-Einträge) — der Owner wählt, wer ausliefert

Der Owner einer Domain wählt den Auslieferungsmodus pro Domain:

- **DHT (Standard)** — ohne NS-Eintrag liefert das Scone-Netzwerk
  die On-Chain-A/AAAA aus (Relay `--dns` oder die NSD-Brücke über
  den Export);
- **unabhängige NS** — der Owner setzt `ns <ip-oder-name>`-Einträge
  (SEINE Nameserver, direkte IP oder Hostname): die Domain ist
  DELEGIERT — das Netzwerk liefert ihren Inhalt nicht mehr aus
  (A/AAAA → NODATA + NS-Autorität), und jeder Auflösende folgt
  den NS des Owners. Die Grenze: **12 unterschiedliche NS pro
  Domain** (alle Formen, nach Wert dedupliziert — das Spiegelbild
  echter Registry-Caps). Das Entfernen eines NS gibt den Slot
  frei; das Entfernen aller stellt den DHT-Modus wieder her.

**Kein Health-Checking existiert**: die Verfügbarkeit der
Endpunkte ist Sache ihrer Betreiber — der Resolver serviert den
signierten Eintragssatz wie er ist, in Eintrags-Reihenfolge (die
Reihenfolge der Einträge ist der Failover-Mechanismus).

### Sicherheit

- GEOROUTE-Einträge sind signierte Einträge wie jeder andere:
  gegen die Chain verifiziert (Owner, Sequenz, record_hash) vor
  jeder Nutzung. **Kein Netzwerk-Vertrauen ohne Validierung** —
  ein unverifizierter DHT-GEOROUTE antwortet nie.
- Die Position des Kunden kommt aus dem **automatisch entdeckten
  Pfad**: das Relay lokalisiert sich selbst über seine externe
  Adresse (STUN/AutoNAT/UPnP), aufgelöst durch die vom Build
  auferlegte eingebettete Tabelle — LAN-Kunden (private Quell-IP)
  erben diese Position, Internet-Kunden werden über ihre eigene
  IP aufgelöst, das Unbekannte fällt auf Welt zurück. Nie aus
  einer Netzwerk-GeoIP-API (kein Abfrage-Leck, keine Abhängigkeit)
  und nie aus einer Betreiber-Erklärung (die Zonen sind vom Build
  auferlegt).
- Die Ziehung ist deterministisch und lokal: kein Geheimnis, keine
  Unvorhersehbarkeit erforderlich (die Sicherheit ist die der
  Eintragssignatur, nicht die der Ziehung).

### Vollständiges Beispiel (Referenzspezifikation)

`www.example.scone` veröffentlicht (Apex `example.scone`, die
ganze Menge bedient jeden Subnamen):

| Scope | Endpunkt | Priorität | Gewicht |
|---|---|---|---|
| US (840) | 10.1.0.1 | 10 | 70 |
| US (840) | 10.1.0.2 | 10 | 30 |
| US (840) | 10.2.0.1 | 20 | 100 |
| TW (158) | 10.3.0.1 | 10 | 80 |
| TW (158) | 10.4.0.1 | 10 | 20 |
| TW (158) | 10.2.0.1 | 20 | 100 |
| WORLD | 10.0.0.1 | 100 | 100 |

Erwartetes Verhalten:

- **US**-Kunde → 10.1.0.1 (≈70%) oder 10.1.0.2 (≈30%); die
  p20-Stufe wird nur erreicht, wenn kein p10-Endpunkt aktiv ist
  (weight = 0 überall) → dann **10.2.0.1**;
- **TW**-Kunde → 10.3.0.1 (≈80%) oder 10.4.0.1 (≈20%);
- **FR**-Kunde (250) → keine FR- oder EU-Route in der Menge →
  WORLD → **10.0.0.1**;
- **A**-Frage auf eine nur-IPv6-Route → leere Antwort (nie ein
  IPv6 in A);
- Domain ohne anwendbare Regel → dieselbe Frage, klassisches
  Verhalten (NODATA: der GEOROUTE hat kein Wire-A-rdata).

### CLI-Optionen (`scone relay`)

| Option | Standard | Rolle |
|---|---|---|
| (keine — GeoDNS immer an, Geo vom Build auferlegt) | an | Automatisches Routing: das Relay entdeckt seine Position (STUN → eingebettete Tabelle), LAN-Kunden erben, Internet-Kunden über ihre IP |
| `--geo-ip-table`, `--geo-config`, `--geo-country/…`, `--health-*` | — | (ENTFERNTE — harter Bruch: die Geozone ist vom Build auferlegt und der Resolver führt keine Health-Prüfungen durch; kein Laufzeit-Flag existiert) |

#### Die eingebettete Tabelle ist die einzige Quelle

Die GeoIP-Tabelle ist **vom Build auferlegt** (DB-IP Lite,
quantisiert, siehe den Eingebettete-Tabelle-Abschnitt oben): keine
Operator-Tabelle kann importiert werden. Auflösung: die
**Quell-IP der DNS-Abfrage** (pro Kunde: UDP `recv_from` / TCP
`peer_addr`) wird in 16 Bytes gegen die eingebettete Tabelle
verglichen; die Regel mit dem **längsten passenden Präfix**
gewinnt (bei Gleichstand die erste in Datei-Reihenfolge).

- **private/nicht routbare Quelle** (RFC1918, ULA fc00::/7,
  link-local, loopback): der Kunde ist IM Netzwerk des Relays —
  er erbt die selbst entdeckte Position des Relays;
- **öffentliche Quelle, abwesend von der Tabelle**: Welt (keine
  Geo-Aufspaltung — nie eine falsche Antwort).

**Strenge** bleibt für jede künftige Tabellen-Oberfläche erhalten:
eine ungültige Zeile ist ein typisierter Start-Fehler, der die
schuldige Zeilennummer trägt; nie eine still ignorierte Zeile, nie
eine Teil-Tabelle.

#### Options-Vorränge

- **Lokalisierer**: die eingebettete Tabelle AUFERLEGT die
  Geolokalisierung — kein Operator-Override. Das Relay entdeckt
  seine Position über seine externe Adresse (STUN/AutoNAT/UPnP,
  durch die Tabelle aufgelöst); LAN-Kunden erben diese Position,
  Internet-Kunden werden über ihre Quell-IP aufgelöst, das
  Unbekannte fällt auf Welt zurück.

Beispiel (Geo-Routing ist aktiv OHNE jedes Geo-Flag):

```bash
scone relay --dns 127.0.0.1:5353
```


## DoH / DoT (verschlüsselte Transporte) — über externes Frontend

Der Scone-DNS-Server spricht **pures UDP und TCP** (RFC 1035 /
7766) — das ist der native Umfang, vollständig und getestet.
Verschlüsselte Transporte zum *Endkunden* (DoH — DNS-over-HTTPS,
RFC 8484; DoT — DNS-over-TLS, RFC 7858) sind an ein
**Standard-Frontend delegiert**: Scone bedient die Daten, das
Frontend bedient den Transport.

Warum dieses Design: ein nativer DoH-Server würde einen vollen
HTTPS-Stapel im Knoten erfordern (Zertifikate, Erneuerung,
HTTP/2) — eine große betriebliche Fläche für null Nutzen, bevor es
browserseitige Endnutzer gibt. Bestehende Frontends tun es besser
und länger; der Knoten bleibt minimal.

### Beispiel: unbound mit DoH vor einem Scone-Relay

`/etc/unbound/unbound.conf`:

```
server:
    interface: 127.0.0.1
    port: 5354
    do-ip4: yes
    do-udp: no
    do-tcp: yes

    # DoH (RFC 8484) auf 443
    https-port: 443
    tls-service-key: "/etc/unbound/scone.key"
    tls-service-pem: "/etc/unbound/scone.pem"

forward-zone:
    name: "."
    forward-addr: 127.0.0.1@5353   # das Scone-Relay (--dns 127.0.0.1:5353)
```

Das Zertifikat (`scone.pem`/`scone.key`) wird wie jeder TLS-Dienst
bezogen — z. B. certbot/acme.sh auf der Domain des Resolvers.
DoH-Kunden zeigen dann auf `https://dns.example.tld/dns-query`.

### Beispiel: knot-resolver mit DoT

`/etc/knot-resolver/kresd.conf`:

```
net.listen('127.0.0.1', 5353, { kind = 'dns' })
net.listen('0.0.0.0', 853, { kind = 'tls' })
tls.cert_file = '/etc/knot-resolver/scone.pem'
tls.key_file  = '/etc/knot-resolver/scone.key'
```

### Was das Frontend hinzufügt

- **Verschlüsselung des Kanals Kunde→Frontend** (das feindselige
  Segment: das Netzwerk des Nutzers); das Segment Frontend→Relay
  bleibt lokal (dieselbe Maschine oder vertrauenswürdiges
  Netzwerk).
- **Einen zusätzlichen Frontend-Cache** (auch unbound/knot-Cache —
  harmlos, die Scone-TTLs bleiben die Obergrenze).
- **Einen stabilen Endpunkt** (IPv4/IPv6, Domain), während Relays
  wechseln.

### Was das Frontend NICHT tut

- Es *verifiziert* nichts: jede vom Relay bediente Antwort ist
  bereits gegen die Chain verifiziert, bevor sie gecacht wird
  (siehe „Auflösungspfad“). Das Frontend relait bereits-autoritative
  Antworten.
- Es ersetzt keine Dezentralisierung: ein DoH-Frontend ist ein
  *betriebener* Dienst — die Wahl des Resolver-Betreibers, nicht
  die des Protokolls. Jedes Scone-Relay bleibt unabhängig über
  natives UDP/TCP nutzbar.

### Künftige native Implementierung (dokumentiert, nicht
zugesagt)

Ein nativer DoH-Transport (`--doh ADDR`) bleibt denkbar: das
Wire-Format ist identisch (RFC 8484 wiederverwendet binäre
DNS-Nachrichten), es wäre ein POST/GET-Dekodieren + HTTP/2 hinter
einem TLS-Proxy (caddy/nginx für Zertifikate), der Knoten selbst
bliebe TLS-frei. Keine Arbeit im Gange — die Nachfrage wird mit
Endnutzern kommen.

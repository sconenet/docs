# CLI (`scone`) — Logging und Ausführlichkeit

> Normatives Verhalten des Loggings der Binärdatei `scone`. Jede
> Abweichung zwischen Dokumentation und Code = Bug (die Quelle der
> Wahrheit beheben).

## Grundsätze

- **stdout ist der Befehlsausgabe vorbehalten** (`scone status`,
  `identity list`, `status`, …). Ein Skript kann stdout umleiten,
  ohne je Logs darin zu finden — Piping bleibt zuverlässig.
  Akzeptierte Ausnahme: `scone relay` gibt die Datenzeilen
  `p2p: <multiaddr>`, `dns: <addr>` und `metrics: <addr>` auf
  stdout aus, sobald die Listener gebunden sind (benutzbare Daten
  für `--bootstrap` / Konfiguration, keine Logs; jeweils genau
  einmal ausgegeben, sofort geflusht).
- **Alle Logs gehen auf stderr** (kompaktes `tracing`-Format),
  einschließlich der des Relays — das Relay loggte auch vor der
  Migration bereits auf stderr.
- Die nutzergerichteten Fehlermeldungen der CLI (`scone: <error>`)
  bleiben auf stderr: das ist UX, keine Logs.

## Stufen und `--verbose` / `-v`

| Aufruf | Effektive Stufe |
|---|---|
| `scone <cmd>` (ohne `-v`) | `warn` |
| `scone relay` (ohne `-v`) | `info` |
| `scone -v <cmd>` / `scone relay -v` | `info` |
| `scone -vv …` | `debug` |
| `scone -vvv …` (und mehr) | `trace` |

`-v`/`--verbose` ist eine **globale und wiederholbare** Flagge: Sie
wird vor oder nach dem Unterbefehl akzeptiert (`scone -vv identity
list` und `scone identity list -vv` sind gleichwertig).

**Warum `scone relay` standardmäßig auf INFO bleibt**: das Relay
ist ein Daemon — es muss seine Aktivität loggen (RPC/p2p-Listening,
angenommene und produzierte Blöcke, verbundene Peers,
Zurückweisungen), um betreibbar zu sein, ohne eine Flagge zu
verlangen. Einmal-Befehle hingegen bleiben still (WARN), um ein
Terminal oder Skript nicht zu verschmutzen.

## `RUST_LOG` (Überschreibung)

Wenn die Umgebungsvariable `RUST_LOG` gesetzt ist, **ersetzt** sie
die aus `-v` abgeleitete Stufe vollständig (`env-filter`-Syntax):

```bash
RUST_LOG=debug scone identity list        # debug ohne -v
RUST_LOG=error scone -vv relay            # error gewinnt über -vv
RUST_LOG=scone_network=trace,info scone relay   # pro Ziel
```

## Umgebungskonfiguration (`.env`)

Vor dem Parsen der Argumente lädt `scone` eine `.env`-Datei aus dem
aktuellen Verzeichnis (`dotenvy::dotenv()` in `main.rs`). Fehlende
Datei = stilles No-Op, nie ein Fehler. Die geladenen Werte werden
die Grundlinie der Relay-Optionen, die eine Umgebungsvariable
deklarieren (`env = "…"` in `cli.rs`).

Vorrang (eine Option, drei Quellen):

```text
explizite CLI-Flagge  >  .env-Variable  >  eingebauter Standardwert
```

- eine auf der Kommandozeile übergebene Flagge GEWINNT IMMER;
- sonst die Variable (ob aus `.env` geladen oder in der echten
  Prozessumgebung gesetzt — dieselbe Variable, dieselbe Wirkung);
- sonst claps eingebauter Standardwert.

Wiederholbare Variablen (`SCONE_BOOTSTRAP`, `SCONE_DNS_UPSTREAM`)
akkumulieren: eine Zeile pro Wert in `.env` (dotenvy hängt an).

Die vollständige Tabelle der `SCONE_*`-Variablen (Netzwerk, P2P
Listen, Bootstrap, Genesis, mDNS, UPnP, DNS-Fläche, Metriken,
Topologie-Profil, Anker) und die Semantik jeder einzelnen lebt in
[`relay.md`](relay.md#umgebungskonfiguration-env); die
kommentierte Vorlage ist `.env.example` an der Repository-Wurzel
(nach `.env` kopieren, nie committen — `.env` ist gitignored).

## Stufen-Semantik (Codekonvention)

- `error!`: Fehlschläge (heute keine Aufrufstellen — Netzwerkfehler
  werden als `warn!` verschluckt, und nur fatale lokale Fehler
  tauchen als typisierter Fehler aus `run()` auf);
- `warn!`: feindselige oder bösartige Netzwerkeinträge abgewiesen,
  alte Txs aus dem Mempool verdrängt, konfliktierende Txs während
  der Produktion verworfen, Aus-/Eingehends-Fehlschläge,
  DHT-Schlüssel-Abweichung;
- `info!`: Block angenommen/produziert, RPC/p2p-Listening, Peer
  verbunden, Relay-Start (Zustands-Checkpoints);
- `debug!`: interne Details (Tx angenommen, Peer getrennt,
  RPC-Roundtrip, Keystore-Auflistung, kad-Ereignisse unter
  `SCONE_KAD_DEBUG`);
- `trace!`: Nutzlasten vorbehalten (heute keine Aufrufstellen).

Bibliotheks-Crates (`scone-network`, …) emittieren
`tracing`-Makros, **initialisieren aber keinen Subscriber**: Die
Initialisierung lebt nur in der Binärdatei `scone` (vor der
Unterbefehls-Dispatch — das Relay braucht seine Logs ab dem Boot).


## Einfache Verwendung (cli-simplify-v1)

- **Kein `--peer` nötig**: Befehle sprechen standardmäßig mit dem
  lokalen Relay; `SCONE_PEER`-Env oder `--peer` überschreibt.
- **Kein `--identity` nötig**, wenn der Keystore genau eine
  Identität hält (sie wird still verwendet; mehrere → der Fehler
  listet sie auf).
- Entfernt (Stubs schlagen die Ersatz vor): `show`, `submit`,
  `lookup`, `owned`.
- Neu: `scone domain renew <name>`, `scone domains`, `scone domain
  get|set` (einzelner Eintrag, keine Datei).

```bash
scone identity generate me
scone domain register example.scone
scone domain set example.scone A 192.0.2.1
scone domain renew example.scone
```

## TLD-Lebenszyklus (cli-unified-lifecycle)

`scone tld register <tld>` ist der Ein-Befehl-TLD-Einstieg,
symmetrisch zu `scone domain register` — der On-Chain-Zustand wird
ZUERST gelesen, dann läuft die richtige Aktion:

- **freie TLD** → registriert, standardmäßig **OFFEN geboren** (keine
  Flagge: Self-Service-Namespace, 1/3 des PoW). `--private` opting
  ein in einen privaten Namespace zum vollen PoW-Preis, mit
  expliziter Bestätigung (wirtschaftliche Entscheidung). `--yes`
  überspringt Bestätigungen (Skripte).
- **Ihre, innerhalb von 90 Tagen ablaufend** → die VERLÄNGERUNG wird
  vorgeschlagen (PoW gebunden an das aktuelle Ablaufdatum —
  Anti-Wiederverwendung) und läuft nach Bestätigung.
- **Ihre, weit vom Ablauf** → nichts zu tun (`tld renew --force`
  verlängert trotzdem).
- **jemand anderem** → sauberer typisierter Fehler mit dem Owner.

```bash
scone tld register mytld            # offene Geburt (Standard)
scone tld register mytld --private  # privater Namespace (bestätigen)
scone tld renew mytld               # explizite Verlängerung
```

Ebenso schlägt `scone domain register <name>` auf einem Namen, der
Ihnen gehört und innerhalb von 90 Tagen abläuft, die Verlängerung
vor, statt zu scheitern.

## Erweiterte Befehle

Die obigen Standards decken den Alltag ab. Für Skripte und
Web/API-Interoperabilität bleibt die vollständige Fläche verfügbar:

- `scone show <name>` — die interne DomainId eines Namens.
- `scone lookup <name>` / `scone owned <pubkey>` — Aliase von
  `domain get` / `domains --pubkey` (gleiche Ausgabe).
- `scone renew <name>` — Alias von `scone domain renew`.
- `scone domain info` — Alias von `scone domain get`.

Die Offline-Hex-Fläche (`tx build|sign|verify`, `submit
tx`-Unterbefehle) wurde ENTFERNT (remove-tx-surface-v2): die
M8b-Operationen, die sie bediente, sind jetzt
Erstklassige-Befehle — `scone tld open <tld>` (einseitige
Self-Service-Öffnung, Öffnungs-PoW für Sie gemint), `scone tld
transfer <tld> <owner>`, `scone tld revoke <tld>`, `scone domain
assign <name> <owner>` (TLD-Eigentümer, private TLD) und `scone
domain transfer <name> <owner>`. Jeder davon:
Zustandslektüre, Signatur, Einreichung, txid. Die Auflösung von
DNS-Einträgen zog zu `scone domain resolve <name>` (ex-`record
get`); der Hex-Veröffentlichungskanal (`record put`) ist weg —
Bearbeitung läuft über `domain set` / `domain update`.
- `scone domain export <name> [--out FILE]` — schreibt das von der
  Chain verifizierte Eintragssatz im exakten `update --file`-Format
  (Roundtrip garantiert: Export → Bearbeitung → Update reicht
  denselben Satz erneut ein). Ohne `--out`: stdout. `domain
  resolve` nutzt dasselbe lesbare Format, sodass `resolve >
  zone.txt` direkt importierbar ist. Arten ohne textuelle Syntax
  (GEOROUTE, unbekannt) werden als kommentierte Warnungen
  exportiert — nie still verworfen.

`domain get` druckt zusätzlich die `domain_id` und `valid_until`.

## Builds: dev vs. mainnet-fähig (mainnet-compile-gate)

Standard-Builds enthalten **kein Mainnet**: `scone relay --network
mainnet` (und Registrar-Dienste mit `SCONE_NETWORK=mainnet`)
scheitern sofort mit einem Rebuild-Hinweis. Das offizielle
verteilte Binary wird gebaut mit:

```bash
cargo build --release --features mainnet
```

`scone --version` zeigt auf solchen Builds `+mainnet`, und das
Relay-Startlog sagt `build: mainnet-enabled`.

Auch der Standard-`--network` folgt dem Build: `mainnet` auf
mainnet-fähigen Builds (das offizielle Binary setzt die Nutzer auf
das Produktionsnetzwerk), sonst `testnet`. Eine explizite
`--network`-Flagge oder das Env `SCONE_NETWORK` gewinnt immer
(clap-Reihenfolge: Flagge > Env > Standard). Das ist eine
betriebliche Schutzleitung (versehentliche Läufe, Dev/CI-Binaries) —
keine kryptografische Grenze; die Konsensregeln bleiben die
Verteidigung gegen feindselige Knoten.

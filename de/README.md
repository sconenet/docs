# Scone-Dokumentation

> Die gesamte Dokumentation unter `docs/` ist **NORMATIV**: Sie muss
> eine unabhängige Implementierung ermöglichen, ohne den Rust-Code zu
> lesen. Jede Abweichung zwischen Dokumentation und Code ist ein Bug
> (die Quelle der Wahrheit ermitteln und beheben).

Zwei Teile:

- **`general/`** — wie es funktioniert: die Konzepte und Regeln des
  Protokolls, verständlich ohne Lektüre der Formatdetails;
- **`technical/`** — detaillierte Spezifikationen für Entwickler:
  exakte Binärformate, Hash-Formeln, Grenzen, APIs und das Verhalten
  jeder Komponente.

## `general/` — wie es funktioniert

| Dokument | Inhalt |
|---|---|
| [architecture.md](general/architecture.md) | Trennung von Autorität (Blockchain) / Verfügbarkeit (DHT) / lokaler Speicherung; Rollen der Crates; DNS-Auflösungsablauf; Entwickelbarkeit |
| [naming.md](general/naming.md) | Namensregeln: LDH-Alphabet, TLDs, Domains und Subdomains, Kanonisierung, `DomainId`-Ableitung und hexadezimale Darstellung |

## `technical/` — Entwicklerspezifikationen

| Dokument | Inhalt |
|---|---|
| [protocol.md](technical/protocol.md) | Kanonisches Binärformat: LEB128-Varints, Bezeichner, DNS-Einträge, Blöcke, P2P-Nachrichten, Grenzen, Versionierung |
| [transactions.md](technical/transactions.md) | Signiertes Transaktionsformat v1: Felder, Wire-Reihenfolge (`RegisterDomain` trägt den Namen, beliebige Diskriminanten), Owner-Ableitung, signierte Nutzlast `SCONE-TX-SIG-V1`, Validierungsregeln |
| [blockchain.md](technical/blockchain.md) | Blockchain-Schicht: `TxId`, Merkle-Baum, `BlockHash`, Genesis, kanonischer Zustand, Blockvalidierung, Konsens-Abstraktion |
| [keystore.md](technical/keystore.md) | `.sconekey`-Schlüsseldateiformat (Argon2id + XChaCha20-Poly1305), `scone-keystore`-API, `identity`-Befehle |
| [storage.md](technical/storage.md) | Lokale redb-Persistenz: Tabellenschema, atomare Deltas, Speicherpolitik, Vertrauensmodell, DoS-Grenzen |
| [relay.md](technical/relay.md) | Netzwerk-Relay (`scone-network`): libp2p-Protokolle, Peer-Erkennung (Seeds, Verzeichnis, mDNS, UPnP), Erreichbarkeit (AutoNAT, Circuit-Relay, DCUtR), Mempool, Synchronisierung, Kademlia-DHT, lokale Topologie, Prometheus-Metriken, DNS-Snapshot, vollständige CLI, `.env`-Konfiguration |
| [dns.md](technical/dns.md) | UDP- + TCP-DNS-Server (M6/M7b): autoritativer DNS-Snapshot, 16-Wege-Shard-Cache im Hot Path (geteilte Einträge, allokierungsfreie Suche, gepackte Einträge, Namenskompression), RFC-1035-Codec, TCP RFC 7766 (512-Byte-UDP-/64-KiB-TCP-Budget, TC-Bit), DHT-Negativcache, rekursiver Fallback, Grenzen |
| [cli.md](technical/cli.md) | `scone`-Binärdatei: Logging und Ausführlichkeit (stdout/stderr, `-v`-Stufen, `RUST_LOG`), Umgebungskonfiguration über `.env` (Vorrang Flagge > `.env` > Standardwert) |
| [security-model.md](technical/security-model.md) | Sicherheitsmodell: Komiteeauswahl, Anti-Grinding der Seeds (Analyse: VRF nicht erforderlich), Äquivokation und Slashing |
| [audit-methodologie.md](technical/audit-methodologie.md) | Methodik des Sicherheitsaudits: Abweichungen, Quelle der Wahrheit, Korrekturdurchgänge |
| [rfc-proof-of-service.md](technical/rfc-proof-of-service.md) | Proof-of-Service-RFC: Heartbeats der Betreiber, Betriebsdelegation, Komiteewahl unter frischen Sitzen |

## Konventionen für Querverweise

Interne Dokumente und Rust-Codekommentare verweisen auf diese Seiten
über absolute Pfade relativ zum Repository-Root
(z. B. `/docs/en/technical/protocol.md`).

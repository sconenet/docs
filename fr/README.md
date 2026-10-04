# Documentation Scone

> Toute la documentation `docs/` est **NORMATIVE** : elle doit permettre une
> implémentation indépendante sans lire le Rust. Toute divergence doc/code
> est un bug (identifier la source de vérité, corriger).

Deux volets :

- **`general/`** — comment ça marche : les concepts et les règles du
  protocole, accessibles sans lire le détail des formats ;
- **`technical/`** — spécifications détaillées pour les développeurs :
  formats binaires exacts, formules de hash, bornes, API et comportement
  de chaque composant.

## `general/` — comment ça marche

| Document | Contenu |
|---|---|
| [architecture.md](general/architecture.md) | Séparation autorité (blockchain) / disponibilité (DHT) / stockage local ; rôles des crates ; flux de résolution DNS ; évolutivité |
| [naming.md](general/naming.md) | Règles de nommage : alphabet LDH, TLD, domaines et sous-domaines, canonicalisation, dérivation et représentation hexadécimale du `DomainId` |

## `technical/` — spécifications développeurs

| Document | Contenu |
|---|---|
| [protocol.md](technical/protocol.md) | Format binaire canonique wire : varints LEB128, identifiants, records DNS, blocs, messages P2P, limites, versionnement |
| [transactions.md](technical/transactions.md) | Format de transaction signée v1 : champs, ordre wire (RegisterDomain porte le nom, discriminants arbitraires), dérivation de l'owner, payload signé `SCONE-TX-SIG-V1`, règles de validation |
| [blockchain.md](technical/blockchain.md) | Couche blockchain : `TxId`, arbre de Merkle, `BlockHash`, genèse, état canonique, validation des blocs, abstraction du consensus |
| [keystore.md](technical/keystore.md) | Format keyfile `.sconekey` (Argon2id + XChaCha20-Poly1305), API `scone-keystore`, commandes `identity` |
| [storage.md](technical/storage.md) | Persistance locale redb : schéma des tables, deltas atomiques, politique mémoire, modèle de confiance, bornes DoS |
| [relay.md](technical/relay.md) | Relay réseau (`scone-network`) : protocoles libp2p, découverte des pairs (graines, annuaire, mDNS, UPnP), joignabilité (AutoNAT, circuit-relay, DCUtR), mempool, sync, DHT Kademlia, topologie locale, métriques Prometheus, snapshot DNS, CLI complet, configuration `.env` |
| [dns.md](technical/dns.md) | Serveur DNS UDP + TCP (M6/M7b) : snapshot DNS autoritaire, cache shardé 16 voies à chemin chaud (entrées partagées, lookup zéro-allocation, records packés, compression de noms), codec RFC 1035, TCP RFC 7766 (budget 512 o UDP / 64 KiB TCP, bit TC), cache négatif DHT, fallback récursif, bornes |
| [cli.md](technical/cli.md) | Binaire `scone` : logging et verbosité (stdout/stderr, niveaux `-v`, `RUST_LOG`), configuration par environnement `.env` (priorité flag > `.env` > défaut) |
| [security-model.md](technical/security-model.md) | Modèle de sécurité : sélection de comité, anti-grinding de la seed (analyse : VRF non nécessaire), équivocation et slashing |
| [audit-methodologie.md](technical/audit-methodologie.md) | Méthodologie d'audit doc/code : divergences, source de vérité, passes de correction |
| [rfc-proof-of-service.md](technical/rfc-proof-of-service.md) | RFC proof-of-service : heartbeat des opérateurs, délégation d'exploitation, élection du comité parmi les sièges frais |

## Conventions de renvoi

Les documents internes et les commentaires du code Rust référencent ces
pages par chemin absolu depuis la racine du dépôt
(ex. `/docs/fr/technical/protocol.md`).

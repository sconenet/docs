# Documentation Scone

> Toute la documentation sous `docs/` est **NORMATIVE** : elle doit
> permettre une implémentation indépendante sans lire le Rust. Toute
> divergence doc/code est un bug (identifier la source de vérité, la
> corriger).

Deux parties :

- **`general/`** — comment ça fonctionne : les concepts et les règles
  du protocole, accessibles sans lire le détail des formats ;
- **`technical/`** — spécifications détaillées pour les développeurs :
  formats binaires exacts, formules de hachage, bornes, API, et le
  comportement de chaque composant.

## `general/` — comment ça fonctionne

| Document | Contenu |
|---|---|
| [architecture.md](general/architecture.md) | Séparation autorité (blockchain) / disponibilité (DHT) / stockage local ; rôles des crates ; flux de résolution DNS ; évolutivité |
| [naming.md](general/naming.md) | Règles de nommage : alphabet LDH, TLD, domaines et sous-domaines, canonicalisation, dérivation du `DomainId` et représentation hexadécimale |

## `technical/` — spécifications développeur

| Document | Contenu |
|---|---|
| [protocol.md](technical/protocol.md) | Format binaire canonique du wire : varints LEB128, identifiants, enregistrements DNS, blocs, messages P2P, limites, versionnage |
| [transactions.md](technical/transactions.md) | Formats de transaction v1 : formes signées (champs, ordre wire — RegisterDomain porte le nom, discriminants arbitraires, dérivation du owner, charge signée `SCONE-TX-SIG-V1`) et les quatre formes sans signature pow-to-owner (la preuve minée EST l'autorisation), la famille à autorité déléguée (liée au compteur : TransferDomain, opérateurs, délégués de records), le compteur de mutations, règles de validation |
| [blockchain.md](technical/blockchain.md) | Couche blockchain : `TxId`, arbre de Merkle, `BlockHash`, genèse, état canonique, validation des blocs, abstraction de consensus |
| [keystore.md](technical/keystore.md) | Format de fichier de clés `.sconekey` (Argon2id + XChaCha20-Poly1305), API `scone-keystore`, commandes `identity` |
| [storage.md](technical/storage.md) | Persistance locale redb : schéma des tables, deltas atomiques, politique mémoire, modèle de confiance, bornes DoS |
| [relay.md](technical/relay.md) | Relais réseau (`scone-network`) : protocoles libp2p, découverte de pairs (seeds, annuaire, UPnP), joignabilité (AutoNAT, circuit-relay, DCUtR), mempool, synchronisation, DHT Kademlia, topologie locale, métriques Prometheus, snapshot DNS, CLI complète, configuration `.env` |
| [dns.md](technical/dns.md) | Serveur DNS UDP + TCP (M6/M7b) : snapshot DNS faisant autorité, cache du chemin chaud partitionné 16 voies (entrées partagées, recherche sans allocation, enregistrements compactés, compression des noms), codec RFC 1035, TCP RFC 7766 (budget 512 octets UDP / 64 KiB TCP, bit TC), cache négatif DHT, repli récursif, bornes |
| [cli.md](technical/cli.md) | Binaire `scone` : logging et verbosité (stdout/stderr, niveaux `-v`, `RUST_LOG`), configuration d'environnement via `.env` (précédence flag > `.env` > défaut), délégués de records (`domain delegates set/clear/list`), updates signées par délégué |
| [security-model.md](technical/security-model.md) | Modèle de sécurité : sélection du comité, anti-grinding des seeds (analyse : VRF non nécessaire), équivoque et slashing |
| [audit-methodologie.md](technical/audit-methodologie.md) | Méthodologie d'audit de sécurité : divergences, source de vérité, passes de correction |
| [rfc-proof-of-service.md](technical/rfc-proof-of-service.md) | RFC preuve-de-service : battements de cœur des opérateurs, délégation d'exploitation, élection du comité parmi les sièges frais |

## Conventions de références croisées

Les documents internes et les commentaires du code Rust référencent ces
pages par chemin absolu depuis la racine du dépôt
(par ex. `/docs/en/technical/protocol.md`).

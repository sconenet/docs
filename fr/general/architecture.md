# Architecture

## Principe directeur

Scone sépare l'autorité de la disponibilité :

- la **blockchain** est la source de vérité pour la propriété et l'ordre des changements ;
- la **DHT** est un système de distribution/réplication, ce n'est **pas** une autorité ;
- le **stockage local** porte l'état du nœud, ses index et ses caches.

Un nœud doit considérer toute donnée venue de la DHT comme potentiellement
malveillante : elle n'est acceptée qu'après vérification cryptographique
(signature du propriétaire, concordance avec l'ownership / sequence /
record_hash de la chaîne).

## Séparation des responsabilités

```text
BLOCKCHAIN
──────────
Ownership
Registration
Authorization
Ordering
Sequence
Record hash
Consensus information

DHT
───
DNS records
Signed records
Replication
Availability

LOCAL STORAGE
─────────────
Local state
Indexes
Cached DHT records
Blockchain state
```

## Crates et dépendances

```text
scone-crypto     ← couche la plus basse : primitives pures sur octets
    ▲
scone-core       ← types du protocole (dépend de scone-crypto)
    ▲
scone-protocol  ← format binaire wire/blockchain
    ▲
scone-blockchain← logique blockchain, état canonique (en mémoire)
    ▲               └ scone-topology (moteur pur, zéro réseau/fs/async ;
                     alimenté par scone-network) — scone-genesis (outil
                     binaire de préparation de la genèse mainnet)
scone-storage   ← persistance locale : backend redb + snapshots vérifiés
    ▲
scone-network   ← relay p2p : libp2p/QUIC, mempool, sync, DHT, DNS
    ▲
scone-keystore  ← keyfiles chiffrés `.sconekey` (dépend de scone-crypto)
    ▲
scone           ← binaire / application
```

### Décision : `scone-core` dépend de `scone-crypto`

Le `DomainId` doit être calculable dès le socle
(`BLAKE3-256(préfixe || nom canonique)`), mais la cryptographie concrète ne
doit pas contaminer le core. Résolution :

- `scone-crypto` est la couche la plus basse : fonctions purement
  déterministes sur octets (plus l'OS RNG pour la génération de clés),
  zéro type du core, aucune dépendance réseau/filesystem/async →
  aucune circularité possible ;
- `scone-core` ne l'utilise que pour le hachage ;
- les signatures Ed25519 et les clés vivent dans `scone-crypto` et
  opèrent sur des octets (payloads canoniques produits par
  `scone-protocol`).

### Rôle de chaque crate

**scone-core** — noms (`TldName`, `DomainName`), identifiants (`DomainId`,
`OwnerId`, `PublicKeyRef`), données DNS (`DnsRecord`, `SignedDnsRecord`),
transactions (`RegisterDomain`, `UpdateDomain`). Déterministe et portable. Interdits
dans ce crate : libp2p, redb, tokio/async, réseau, fichiers, OS, serveur
DNS, logique blockchain.

**scone-crypto** — primitives cryptographiques. Aujourd'hui : `hash256`
(BLAKE3), signatures et clés Ed25519 (`SigningKey`,
`PublicKey`, `Signature`, vérification groupée stricte), enveloppe AEAD
de groupes de records (XChaCha20-Poly1305). Prévu : Merkle tree.
Règle : aucun algorithme maison, dépendances auditées uniquement.

**scone-protocol** — format binaire du protocole : encodage/décodage
canonique (varint LEB128 minimal, identifiants 32 octets nus) des
transactions, records DNS, blocs (`Block`/`BlockHeader` au champ
consensus opaque) et messages P2P, avec limites de taille vérifiées
avant allocation, rejet des formes non canoniques et versionnement
(`PROTOCOL_VERSION`). Aucune logique de consensus, de réseau ou de
stockage. Référence du format : `/docs/fr/technical/protocol.md`.

**scone-blockchain** — logique blockchain et état canonique :
identification des transactions (`TxId`), arbre de Merkle ordonné,
`BlockHash`, genèse déterministe, validation des blocs et transactions,
règles d'application à l'état (`DomainState`/`ChainState`, en mémoire,
accès direct par `DomainId`), détection de forks et choix de branche
(`fork_choice` : plancher de finalité + chaîne la plus longue +
tie-break par hash), finalité par checkpoints et abstraction du
consensus (`trait Consensus`, impl permissive par défaut). Le PoW
d'enregistrement (mine/verify, difficultés par réseau) vit dans
`scone-core`. Interdits dans ce crate : réseau, DHT, stockage (redb),
serveur DNS. Référence :
`/docs/fr/technical/blockchain.md`.

**scone-storage** — persistance locale (livrée). Le backend concret est
`redb` (`NodeStore`), caché derrière un trait minimal afin que
blockchain/DHT/DNS ne soient jamais couplés directement à redb ; un autre
backend (KV distribué) pourra être branché. Inclut le backend d'état KV
paginé, le journal de blocs, les snapshots vérifiés (manifeste + pages
hachés) et le chemin de restauration/replay après redémarrage. Référence :
`/docs/fr/technical/storage.md`.

**scone-network** — relay p2p du nœud : swarm libp2p/QUIC, relais de
transactions et blocs (validation locale systématique), mempool borné,
sync de blocs bornée (128 blocs/requête), DHT Kademlia pour les `SignedDnsRecord`, serveur DNS UDP+TCP
(GeoDNS toujours actif — table GeoIP embarquée, fallback
récursif), boucle checkpoint/heartbeat des anchors (le CLI parle
QUIC p2p direct au relay, plus de RPC local). Joignabilité et
découverte : AutoNAT v2 (verdict reachable/NATé), circuit-relay v2
+ DCUtR pour les nœuds NATés, mDNS (découverte LAN zéro-config,
armée par défaut sur devnet) et
UPnP (mapping du port QUIC par le routeur, opt-in).
Configuration par fichier `.env` (voir `.env.example`).
Référence : `/docs/fr/technical/relay.md`.

**scone-keystore** — stockage chiffré des clés de signature Ed25519
(fichiers `.sconekey` : Argon2id + XChaCha20-Poly1305). Ce crate prend
en charge le filesystem pour que `scone-crypto` reste pur. Référence :
`/docs/fr/technical/keystore.md`.

**scone-topology** — moteur PUR d'optimisation locale de topologie p2p :
zéro réseau, zéro filesystem, zéro async. Ne connaît ni libp2p ni le
swarm : `scone-network` l'alimente en observations (RTT ping,
connexions) et reçoit des décisions de gestion de voisinage.

**scone-genesis** — binaire outil (kickstarter) de préparation de la
genèse mainnet : manifeste JSON versionné, allocations, bloc
déterministe.

**scone** — binaire/CLI. Ne dépend d'une crate que lorsqu'il l'utilise
réellement (aujourd'hui : scone-network pour le relay, scone-blockchain,
scone-storage, scone-keystore, scone-protocol, scone-core,
scone-crypto, scone-topology).

## Le nœud

Un nœud Scone combine :

1. une blockchain (validation du consensus, état d'ownership) ;
2. un routage DHT (publication/récupération des `SignedDnsRecord`) ;
3. un stockage local (état, index, caches) ;
4. un resolver / serveur DNS en façade.

### Démarrage d'un nouveau nœud (bootstrap d'état, P0.3/P0.4)

Un nouveau nœud ne rejoue PAS la genèse : il démarre d'un état
finalisé vérifié.

```text
finalized checkpoint (comité, quorum de signatures — validé par
        │              la couche chaîne avant tout usage)
        ▼
verified snapshot (manifest + pages paginées ; l'importateur
        │              RECALCULE la racine SMT depuis les entrées
        │              et la compare au state_root signé — un
        │              snapshot falsifié/incomplet est rejeté
        │              avant toute écriture)
        ▼
bloc d'ancrage H (demandé au réseau PAR HASH)
        │
        ▼
blocs récents H+1..H+k (relay, hauteur croissante)
        │
        ▼
synced — au boot, rejeu du SEUL suffixe au-dessus de H
```

Deux profils de nœuds :

- **nœud archive** (drapeau opérateur) : garde tous les blocs
  historiques, les sert au réseau, démarre par les chemins
  existants ;
- **nœud full-state** : détient l'état complet à H et le suffixe,
  mais pas l'historique sous H — il valide les blocs futurs et
  sert l'état, il ne sert pas l'historique.

Ce chemin est OPT-IN : l'import d'un snapshot vérifié est une
action explicite de l'opérateur (`import_verified_snapshot`),
jamais automatique — le comportement par défaut d'un nœud existant
ne change pas. Détails normatifs : `/docs/fr/technical/storage.md`,
sections « Snapshot d'état VÉRIFIÉ » et « Bootstrap d'état ».

## Flux de résolution DNS

```text
requête DNS (example.uip)
    │
    ▼
Node : DomainId = BLAKE3-256("SCONE-DOMAIN-V1" || "example.uip")
    │
    ├── snapshot DNS autoritaire (RAM) ──► inconnu/expiré → NXDOMAIN
    │                                         (sans toucher au cœur)
    ├── cache DNS shardé (qname,qtype) ───► hit : entrée partagée,
    │                                         vérification paresseuse
    ├── cache local (SignedDnsRecord) ────► vérification (signature, owner,
    │                                         sequence, record_hash vs chaîne)
    ├── DHT get(DomainId) ────────────────► même vérification
    │                                         (miss → cache négatif 30 s)
    ▼
réponse DNS (si et seulement si valide)
```

Le snapshot DNS (existence + échéance de chaque domaine vivant,
publié à chaque bloc appliqué) garantit qu'une décision
d'existence ne traverse jamais le run loop du relay : le cœur
n'est sollicité que pour le fetch effectif des records.

## Évolutivité

- préimages versionnées (`SCONE-DOMAIN-V1`, `SCONE-PUBKEY-V1`,
  `SCONE-OWNER-V1`, …) : changer une règle de dérivation crée un nouvel
  espace de noms plutôt que de casser l'existant ;
- `RecordData` extensible par variants + `Unknown { type_code, data }`
  pour les types non connus à la lecture ;
- `PROTOCOL_VERSION` (`scone-protocol`) pour les ruptures de format wire ;
- la chaîne ne stocke que des hashs : les données volumineuses restent
  en DHT.

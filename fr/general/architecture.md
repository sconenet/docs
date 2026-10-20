# Architecture

## D'abord en mots simples

Cinq concepts suffisent pour tout comprendre :

1. **Un nom** — une adresse sur internet, comme `mysite.scone`.
2. **Votre phrase de 24 mots** — votre identité. C'est le titre de
   propriété qui prouve qu'un nom est le vôtre, et qui signe chaque
   changement que vous effectuez.
3. **Le registre public** (la blockchain) — le grand livre partagé et
   public de qui possède quoi. Personne ne le contrôle, personne ne
   peut le réécrire.
4. **Le minage** — la façon de réserver un nom : votre ordinateur
   effectue un travail, équitable et identique pour tous. Pas de
   file d'attente, pas d'intermédiaire.
5. **Un relais** — un ordinateur qui fait tourner le réseau en
   gardant une copie du registre et en répondant aux questions.
   N'importe qui peut en faire tourner un ; plus il y en a, plus le
   réseau est solide.

Le reste de cette page approfondit — vous pouvez vous arrêter ici et
déjà comprendre le projet.

## Principe directeur

Scone sépare l'autorité de la disponibilité :

- la **blockchain** est la source de vérité pour la propriété et
  l'ordre des changements ;
- la **DHT** est un système de distribution/réplication, elle n'est
  **pas** une autorité ;
- le **stockage local** contient l'état du nœud, les index et les
  caches.

Un nœud doit traiter toute donnée venant de la DHT comme
potentiellement malveillante : elle n'est acceptée qu'après
vérification cryptographique (signature du propriétaire,
correspondance avec le propriétaire / la séquence / le record_hash de
la chaîne).

## Séparation des responsabilités

```text
BLOCKCHAIN
──────────
Propriété
Enregistrement
Autorisation
Ordre
Séquence
Hash d'enregistrement
Informations de consensus

DHT
───
Enregistrements DNS
Enregistrements signés
Réplication
Disponibilité

STOCKAGE LOCAL
──────────────
État local
Index
Enregistrements DHT en cache
État de la blockchain
```

## Crates et dépendances

```text
scone-crypto     ← couche la plus basse : primitives pures sur des octets
    ▲
scone-core       ← types du protocole (dépend de scone-crypto)
    ▲
scone-protocol  ← format binaire wire/blockchain
    ▲
scone-blockchain← logique blockchain, état canonique (en mémoire)
    ▲               └ scone-topology (moteur pur, zéro réseau/fs/
                     async ; alimenté par scone-network) — scone-genesis
                     (outil binaire one-shot de genèse mainnet)
scone-storage   ← persistance locale : backend redb + snapshots vérifiés
    ▲
scone-network   ← relais p2p : libp2p/QUIC, mempool, sync, DHT, DNS
    ▲
scone-keystore  ← fichiers de clés chiffrés `.sconekey` (dépend de scone-crypto)
    ▲
scone           ← binaire / application
```

### Décision : `scone-core` dépend de `scone-crypto`

Le `DomainId` doit être calculable dès la fondation
(`BLAKE3-256(prefix || canonical name)`), mais la cryptographie
concrète ne doit pas contaminer le core. Résolution :

- `scone-crypto` est la couche la plus basse : fonctions purement
  déterministes sur des octets (plus le RNG de l'OS pour la génération
  de clés), zéro type du core, aucune dépendance réseau/système de
  fichiers/async → aucune circularité possible ;
- `scone-core` ne l'utilise que pour le hachage ;
- les signatures et clés Ed25519 vivent dans `scone-crypto` et
  opèrent sur des octets (charges canoniques produites par
  `scone-protocol`).

### Rôle de chaque crate

**scone-core** — noms (`TldName`, `DomainName`), identifiants
(`DomainId`, `OwnerId`, `PublicKeyRef`), données DNS (`DnsRecord`,
`SignedDnsRecord`), transactions (`RegisterDomain`, `UpdateDomain`).
Déterministe et portable. Interdits dans cette crate : libp2p, redb,
tokio/async, réseau, fichiers, OS, serveur DNS, logique blockchain.

**scone-crypto** — primitives cryptographiques. Aujourd'hui :
`hash256` (BLAKE3), signatures et clés Ed25519 (`SigningKey`,
`PublicKey`, `Signature`, vérification stricte par lots), enveloppe
AEAD des groupes d'enregistrements (XChaCha20-Poly1305). Prévu :
arbre de Merkle. Règle : aucun algorithme maison, dépendances
auditées uniquement.

**scone-protocol** — format binaire du protocole : encodage/décodage
canonique (varint LEB128 minimal, identifiants de 32 octets nus) des
transactions, enregistrements DNS, blocs (`Block`/`BlockHeader` avec
un champ de consensus opaque) et messages P2P, avec limites de taille
vérifiées avant allocation, rejet des formes non canoniques et
versionnage (`PROTOCOL_VERSION`). Aucune logique de consensus, de
réseau ou de stockage. Référence du format :
`/docs/en/technical/protocol.md`.

**scone-blockchain** — logique blockchain et état canonique :
identification des transactions (`TxId`), arbre de Merkle ordonné,
`BlockHash`, genèse déterministe, validation des blocs et des
transactions, règles d'application de l'état (`DomainState`/
`ChainState`, en mémoire, accès direct par `DomainId`), détection de
forks et choix de branche (`fork_choice` : plancher de finalité +
chaîne la plus longue + départage par hash le plus bas), finalité des
points de contrôle, et abstraction de consensus (`trait Consensus`,
implémentation permissive par défaut). Le PoW d'enregistrement
(miner/vérifier, difficultés par réseau) vit dans `scone-core`.
Interdits dans cette crate : réseau, DHT, stockage (redb), serveur
DNS. Référence : `/docs/en/technical/blockchain.md`.

**scone-storage** — persistance locale (livrée). Le backend concret
est `redb` (`NodeStore`), masqué derrière un trait minimal pour que
blockchain/DHT/DNS ne soient jamais directement couplés à redb ; un
autre backend (KV distribué) pourrait être branché. Comprend le
backend d'état KV paginé, le journal de blocs, les snapshots vérifiés
(manifeste haché + pages) et le chemin de restauration/rejeu au
redémarrage. Référence : `/docs/en/technical/storage.md`.

**scone-network** — le relais p2p du nœud : swarm libp2p/QUIC, relais
de transactions et de blocs (validation locale systématique), mempool
borné, synchronisation de blocs bornée (128 blocs/requête), DHT
Kademlia pour `SignedDnsRecord`, serveur DNS UDP+TCP (GeoDNS toujours
actif — table GeoIP embarquée, repli récursif), la boucle de point
de contrôle/battement de cœur des ancrès (le CLI parle directement
en p2p QUIC au relais, plus de RPC local). Joignabilité et découverte :
AutoNAT v2 (verdict joignable/sous NAT), circuit-relay v2 + DCUtR
pour les nœuds sous NAT, mDNS (découverte LAN sans configuration,
ACTIVE par défaut sur devnet) et UPnP (mappage du port QUIC par le
routeur, opt-in). Configuration via un fichier `.env`
(voir `.env.example`). Référence : `/docs/en/technical/relay.md`.

**scone-keystore** — stockage chiffré des clés de signature Ed25519
(fichiers `.sconekey` : Argon2id + XChaCha20-Poly1305). Cette crate
possède le système de fichiers pour que `scone-crypto` puisse rester
pur. Référence : `/docs/en/technical/keystore.md`.

**scone-topology** — moteur PUR d'optimisation de la topologie p2p
locale : zéro réseau, zéro système de fichiers, zéro async. Il ne
connaît ni libp2p ni le swarm : `scone-network` l'alimente en
observations (RTT des pings, connexions) et reçoit des décisions de
gestion de voisinage.

**scone-genesis** — outil binaire one-shot (kickstarter) pour la
préparation de la genèse mainnet : manifeste JSON versionné,
allocations, bloc déterministe. L'outil produit les artefacts, le
dépôt ne les versionne jamais (refus d'écriture dans un workspace
Rust ; distribution réservée au lancement, cf. `blockchain.md`).

**scone** — binaire/CLI. Ne dépend d'une crate que lorsqu'il
l'utilise réellement (aujourd'hui : scone-network pour le relais,
scone-blockchain, scone-storage, scone-keystore, scone-protocol,
scone-core, scone-crypto, scone-topology).

## Le nœud

Un nœud Scone combine :

1. une blockchain (validation de consensus, état de propriété) ;
2. du routage DHT (publication/récupération des `SignedDnsRecord`) ;
3. du stockage local (état, index, caches) ;
4. un résolveur / serveur DNS au front.

### Démarrer un nouveau nœud (amorce de l'état, P0.3/P0.4)

Un nouveau nœud ne rejoue PAS la genèse : il démarre depuis un état
finalisé vérifié.

```text
point de contrôle finalisé (comité, quorum de signatures — validé par
        │              la couche chaîne avant tout usage)
        ▼
snapshot vérifié (manifeste + pages paginées ; l'importeur
        │              RECALCULE la racine SMT depuis les entrées
        │              et la compare au state_root signé —
        │              un snapshot falsifié/incomplet est rejeté
        │              avant toute écriture)
        ▼
bloc d'ancrage H (demandé au réseau PAR HASH)
        │
        ▼
blocs récents H+1..H+k (relais, hauteur croissante)
        │
        ▼
synchronisé — au démarrage, rejeu du SEUL suffixe au-dessus de H
```

Deux profils de nœuds :

- **nœud d'archive** (flag opérateur) : garde tous les blocs
  historiques, les sert au réseau, démarre via les chemins existants ;
- **nœud à état complet** : détient l'état complet à H et le suffixe,
  mais pas l'historique en dessous de H — il valide les blocs futurs
  et sert l'état, il ne sert pas l'historique.

Ce chemin est OPT-IN : importer un snapshot vérifié est une action
explicite de l'opérateur (`import_verified_snapshot`), jamais
automatique — le comportement par défaut d'un nœud existant ne change
pas. Détails normatifs : `/docs/en/technical/storage.md`, sections
« VERIFIED state snapshot » et « State bootstrap ».

## Flux de résolution DNS

```text
requête DNS (example.uip)
    │
    ▼
Nœud : DomainId = BLAKE3-256("SCONE-DOMAIN-V1" || "example.uip")
    │
    ├── snapshot DNS faisant autorité (RAM) ─► inconnu/expiré → NXDOMAIN
    │                                          (sans toucher le core)
    ├── cache DNS partitionné (qname,qtype) ─► touché : entrée partagée,
    │                                          vérification paresseuse
    ├── cache local (SignedDnsRecord) ──────► vérification (signature,
    │                                          propriétaire, séquence,
    │                                          record_hash vs chaîne)
    ├── DHT get(DomainId) ─────────────────► même vérification
    │                                          (échec → cache négatif 30 s)
    ▼
réponse DNS (si et seulement si valide)
```

Le snapshot DNS (existence + expiration de chaque domaine vivant,
publié après chaque bloc appliqué) garantit qu'une décision
d'existence ne traverse jamais la boucle d'exécution du relais : le
core n'est touché que pour les récupérations effectives
d'enregistrements.

## Évolutivité

- préimages versionnées (`SCONE-DOMAIN-V1`, `SCONE-PUBKEY-V1`,
  `SCONE-OWNER-V1`, …) : changer une règle de dérivation crée un
  nouvel espace de noms plutôt que de casser l'existant ;
- `RecordData` extensible via des variantes + `Unknown { type_code, data }`
  pour les types inconnus au moment de la lecture ;
- `PROTOCOL_VERSION` (`scone-protocol`) pour les ruptures du format
  wire ;
- la chaîne ne stocke que des hashs : les données volumineuses restent
  dans la DHT.

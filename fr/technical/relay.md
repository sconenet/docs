# Relais réseau (M4) — architecture, flux et contrôle CLI

> Spécification normative du relais réseau Scone (`scone-network`).
> Toute divergence doc/code = bug (corriger la source de vérité).

## Vue d'ensemble

Le **relais** est le nœud P2P complet de Scone : il maintient la
blockchain canonique (RAM + `NodeStore` redb), relaie les transactions
et les blocs, produit des blocs (mode devnet), sert le protocole de
synchronisation, publie et résout les `SignedDnsRecord`s dans une DHT
Kademlia, et expose le **protocole p2p** (`/scone/reqres/1`) que la CLI
interroge directement (unification RPC→p2p : le canal TCP JSON a
disparu).

```text
                    ┌────────────────────────────────────────┐
                    │              scone relay               │
  P2P (libp2p)      │                                        │
  QUIC (udp,        │  Swarm libp2p                          │
  port éphémère     │  ├─ identify  (découverte d'adresses)  │
  par défaut) ◄─────┤  ├─ ping      (vérification de vie)    │
                    │  ├─ kad       (DHT d'enregistrements)  │
                    │  └─ reqres    (messages scone-prot.)   │
                    │                                        │
                    │  Cœur du relais (tâche unique)         │
                    │  ├─ Blockchain (validation complète)   │
                    │  ├─ Mempool (borné, dédup par TxId)    │
                    │  ├─ BlockBuilder (production devnet)   │
                    │  ├─ Parking de forks (borné, 2048)     │
                    │  └─ Pilote de sync (rattrapage par lots)│
                    │                                        │
  CLI ◄─────────────┤  reqres p2p (QUIC, mêmes messages)     │
                    └────────────────┬───────────────────────┘
                                     │
                              NodeStore (redb)
```

`crates/scone-network` est la **seule crate async** du workspace :
tout ce qui est en dessous (crypto → core → protocol → blockchain →
storage) reste synchrone et pur.

## Admission des connexions (P1.5)

Limites dures au niveau du swarm (comportement `connection_limits` de
libp2p, première entrée de `SconeBehaviour`) :

- `MAX_TOTAL_CONNECTIONS = 128` — plafond de connexions établies
  (entrantes + sortantes) par relais ; l'excédent entrant est REFUSÉ,
  jamais mis en file ;
- `MAX_PER_PEER_CONNECTIONS = 4` — un seul pair ne peut pas monopoliser
  le quota total en ouvrant de nombreuses connexions ;
- `MAX_PENDING_INCOMING = 32` — garde slowloris : les demi-connexions
  non authentifiées n'attendent jamais indéfiniment.

Ces limites complètent les limites par message existantes
(`MAX_MESSAGE_LEN` 1 MiB) et les limites par budget (sync `max ≤ 128`
blocs, DNS TCP 64 connexions/512 B/5 s). Un attaquant par inondation de
connexions reçoit des refus, pas une croissance mémoire.

## Découverte des pairs

Trois mécanismes complémentaires, dans cet ordre de priorité :

1. **Seeds explicites** (`--bootstrap`, multiaddrs complètes) — toujours
   prioritaires ; fournir une liste non vide REMPLACE tout le reste.
   `--no-builtin-seeds` vide les seeds embarquées pour des déploiements
   100 % privés.
2. **Seeds embarquées** (`builtin_seeds(network)` dans
   `scone-network/src/config.rs`) — figées à chaque publication
   officielle de binaire, par réseau. Un relais non configuré rejoint
   seul le réseau. (Vides sur testnet tant que le testnet public n'est
   pas déployé.)
3. **Annuaire appris** (`<data-dir>/peers.json`) — toute adresse
   apprise via `identify` est conservée (vue en dernier ; plafond de
   **2000 entrées** — des adresses, pas des connexions, ~400 KiB de RAM
   au maximum — et une **coupe à 90 jours** : une entrée vue pour la
   dernière fois il y a plus de 90 jours est abandonnée à l'écriture et
   ignorée au chargement, gardant les 2000 slots remplis de pairs
   plausibles plutôt que de fossiles pré-churn dont la seule
   contribution serait un timeout de numérotation au démarrage) et
   sauvegardée ~1×/min au tick de production. Au démarrage sans seed et
   sans bootstrap, l'annuaire est recomposé : le relais retrouve ses
   pairs habituels sans configuration. C'est un cache : fichier
   manquant/corrompu = annuaire vide, jamais une erreur.
4. **mDNS LAN** (`--mdns`) — découverte sans configuration sur le lien
   local : les relais voisins d'un même segment réseau se trouvent SANS
   aucune seed (le chemin devnet zéro-configuration). **Défaut :
   ACTIF sur devnet, INACTIF ailleurs** (un réseau local réinitialisable
   est un réseau de découverte par conception ; un déploiement public ne
   multidiffuse rien). Comportement : bind de la socket multicast ; un
   échec de bind (pas de multicast sur l'interface, environnement
   bac à sable) **désactive** mDNS avec un avertissement — jamais fatal
   (la découverte est un confort, jamais une bouée de sauvetage).
   Chaque voisin `Discovered` alimente la table Kademlia
   (`kad.add_address`), exactement comme un bootstrap explicite moins
   la configuration ; les entrées `Expired` sont suivies par libp2p
   lui-même (la connexion se ferme à son propre timeout). Les membres
   `mdns`/`upnp` du behaviour sont des `Toggle`s (activés/désactivés à
   la construction,
   `build_behaviour(key, enable_mdns, enable_upnp)`).
5. **UPnP** (`--upnp`, opt-in) — mappage du port d'écoute QUIC par la
   box domestique : le nœud devient joignable depuis l'extérieur SANS
   circuit relay. Sur `NewExternalAddr`, l'adresse externe est
   enregistrée comme vraie adresse externe du swarm
   (`add_external_address` — vue par identify/autonat, candidate
   transport-relay). Pas de passerelle IGD (`GatewayNotFound`) ou
   passerelle non routable (`NonRoutableGateway`) : un simple log
   d'info, le mappage est sauté — cas normaux sur un LAN domestique. Un
   mappage expiré (échec de renouvellement) est un avertissement.

mDNS suit le réseau (un `--mdns`/`SCONE_MDNS` explicite gagne
toujours) : **ACTIF par défaut sur devnet**, INACTIF sur
testnet/mainnet ; UPnP reste **désactivé par défaut**
(`enable_mdns`/`enable_upnp` = `false` dans `config.rs`, le relais
active mDNS pour devnet) : un déploiement public ne multidiffuse
rien et ne parle à aucun routeur.

### Optimisation de la topologie locale (toujours active)

Le relais optimise activement son voisinage via le moteur pur
`scone-topology` (voir la doc de la crate). **Toujours activé** ; le
seul réglage exposé est un **profil** (`--topology-profile`, env
`SCONE_TOPOLOGY_PROFILE`, défaut `default`) :

| Profil | min / cible / max | éval. | pour qui |
|---|---|---|---|
| `tiny` | 2 / 4 / 6 | 120 s | Raspberry Pi, box domestique, VPS 1 vCPU |
| `small` | 4 / 8 / 12 | 90 s | petit serveur, NAS domestique |
| `default` | 8 / 16 / 32 | 60 s | serveur standard (défaut) |
| `high` | 16 / 32 / 64 | 45 s | nœud souverain, backbone |
| `very-high` | 32 / 64 / 128 | 30 s | hubs fondateurs |
| `devtest` | 2 / 4 / 6 | 5 s | **LABO/TEST uniquement** (instable par conception) |

```
scone relay --topology-profile tiny
```

Les paramètres protecteurs (hystérésis 0,15, cooldown 600 s, grâce
120 s, mesure 15 s) sont identiques partout : ils gardent la stabilité
et n'ont aucune raison de passer à l'échelle de la taille du nœud. Un
nom inconnu est une erreur typée au démarrage.

Une configuration incohérente (`min > cible > max`, seuil hors
`[0,1]` — `NaN` inclus —, intervalle nul) est une **erreur typée au
démarrage**, jamais un comportement silencieux.

**Vue (`TopologyView`)** — alimentée par les événements du swarm :

- `ping` réussi → observation RTT (fenêtre médiane de 8 échantillons) ;
- `ping` échoué → pénalité de fiabilité ;
- `identify` reçu → adresses (≤ 4/pair) + protocoles annoncés
  (masque kad/reqres/identify/relay) ;
- connexion établie → état connecté, direction
  (entrante/sortante), âge, circuit-relay détecté ;
- connexion fermée → pénalité + compteur de churn SEULEMENT si non
  sollicitée (une éviction volontaire ne compte pas) ;
- échec de numérotation sortante → pénalité de fiabilité.

La vue est bornée (500 pairs ; au-delà, seuls les pairs connectés sont
mémorisés).

**`kad_required` depuis les kbuckets** — avant chaque évaluation, le
relais parcourt la table de routage Kademlia : un pair est protégé s'il
est `Connected` dans un bucket, ou `Disconnected` dans un bucket
SATURÉ (k = 20). Un pair protégé n'est JAMAIS choisi pour éviction.

**Boucle d'optimisation** — à l'intervalle d'évaluation du profil
(60 s sous `default` ; voir le tableau ci-dessus) :

1. recalcul de `kad_required` (kbuckets) ;
2. évaluation du moteur (hystérésis : période de grâce de 120 s,
   amélioration minimale du score de +15 %, cooldown d'éviction
   600 s) ;
3. décision appliquée : `AddCandidate` → numérotation du candidat ;
   `Replace` → fermeture du pair évincé (UNE FOIS — progressif, pas de
   bouclage) puis numérotation du remplaçant ;
4. un instantané de **métriques** est journalisé (niveau INFO) :
   comptages connus/connectés/entrants/sortants/circuit/protégés,
   RTT moyen/médian, buckets IP distincts, concentration maximale,
   churn (remplacements/déconnexions).

Le score de voisinage n'est JAMAIS une mesure de confiance (voir la
doc de `scone-topology`) : aucune décision de consensus n'en dérive.

### Joignabilité (AutoNAT) et transport relay (circuit-relay v2)

Chaque relais exécute AutoNAT (client + serveur) : le nœud demande à
ses pairs de tenter une numérotation RETOUR vers lui ; le verdict
(`reachable` : oui = joignable depuis l'extérieur, non = derrière NAT,
inconnu = pas encore testé) est rafraîchi toutes les 5 min et exposé
dans `NodeStatus` (champ `nat_reachable`, wire varint 0/1/2) — la CLI
l'affiche (`scone status`). Les adresses privées comptent
(`only_global_ips = false`) : un maillage LAN teste sa joignabilité
LAN.

Un relais joignable active le **hop** circuit-relay v2 (il peut relayer
du trafic pour des pairs NATés) et **DCUtR** (hole-punching : deux
nœuds connectés via circuit tentent une connexion directe et libèrent
le relais). Un relais jugé NATé demande automatiquement une réservation
au premier pair joignable connu et écoute à travers lui (adresse
`/p2p/<relay>/p2p-circuit`).

Règle de production : un nœud NATé relayé ne devrait pas devenir
producteur de blocs (latence/disponibilité dégradées) — la règle PoS
(siège = domaine vivant) ne l'interdit pas encore ; à documenter comme
exigence opérationnelle avant le testnet public.

## Transport et protocoles libp2p

| Protocole | Nom | Rôle |
|---|---|---|
| QUIC (quic-v1) | — | Transport principal (udp) |
| identify | `/scone/id/1` | Découverte d'adresses des pairs (remplit la table Kademlia) |
| ping | standard libp2p | Vérification de vie |
| Kademlia | `/scone/kad/1` | DHT : clé = `DomainId` (32 B), valeur = encodage canonique du `SignedDnsRecord` |
| request-response | `/scone/reqres/1` | messages applicatifs scone-protocol |

### Trame applicative (`/scone/reqres/1`)

Chaque trame est `len: u32 BE || bytes` où les bytes sont l'encodage
canonique d'un `Message` de `scone-protocol` (le codec existant — pas
de second format). La longueur est bornée par
`limits::MAX_MESSAGE_LEN` **avant** toute lecture ou allocation : un
pair annonçant une trame plus grande est rejeté sans coût mémoire.

Messages échangés (un par aller-retour requête/réponse) :

| Message | Direction | Réponse |
|---|---|---|
| `Hello { version }` | poignée de main | `Hello` (version locale) |
| `Ping` / `Pong` | vérification de vie | `Pong` |
| `GetBlock { hash }` | requête | `Block` (fenêtre RAM) ou `Pong` si inconnu |
| `GetBlocks { start, max }` | sync | **un** `Block` (hauteur `start`), ou `Pong` si aucun |
| `Block (b)` | relais de bloc | `Pong` (ack) |
| `Transaction (tx)` | relais de tx | `Pong` (ack) |
| `GetRecord { domain_id }` | DHT local | `Record` (cache local) ou `Pong` |
| `Record (r)` | publication | `Pong` si accepté, erreur sinon |

Règle : **exactement une réponse par requête** ; les messages
d'accusé de réception sont des `Pong`. Les réponses de sync portent un
bloc à la fois — le rattrapage itère côté demandeur (voir « Sync »).
C'est délibérément borné : jamais plus d'un bloc en vol par pair.

## Comportement du relais

### Démarrage

0. résolution du réseau (`--network testnet|mainnet|devnet`, testnet par
   défaut tant que le projet est en dev — M8b). Le data-dir est **par
   réseau** (`~/.scone/<network>/`) : un store non vide contenant une
   chaîne d'un autre réseau est refusé (`Corrupted`) ;
1. ouverture/création du store redb (`<data-dir>/chain.redb`) ;
2. `load_chain(store, network)` : sommet O(1) + états de domaines
   paginés (fenêtre RAM : genèse + sommet) ;
3. construction du swarm (identité Ed25519 éphémère par session —
   rendre persistante l'identité réseau est une décision ouverte) ;
4. composition de la liste de démarrage (voir
   [Découverte des pairs](#découverte-des-pairs)) : `--bootstrap`
   explicite (doit finir par `/p2p/<peer-id>`) SINON les seeds
   embarquées du réseau SINON l'annuaire appris (`peers.json`) ;
5. numérotation de chaque multiaddr et inscription dans la table
   Kademlia (une numérotation morte est ignorée — les autres font le
   réseau).

### Réception d'une transaction (soumission P2P ou CLI `submit tx`)

1. décodage strict borné (`MAX_MESSAGE_LEN`) ;
2. validation cryptographique complète (`validate_transaction` :
   liaison propriétaire/clé, `verify_strict` sur la charge canonique) ;
3. **anti-replay D'ABORD** (bon marché, indépendant de l'état,
   définitif), puis pré-contrôle d'état (miroir bon marché des règles
   de `push_block` — la PoW, coûteuse, n'est vérifiée qu'au push) : un
   `RenewDomain` rejoué répond `TxReplay`, pas une erreur de PoW
   mensongère (sa preuve lie l'expiration que la première application
   a dépassée). La séparation de réseau est appliquée à l'étape 2 : la
   charge signée injecte le réseau du RELAIS, une tx signée ailleurs
   échoue avec `InvalidSignature` (wire 2026-09 : plus de champ
   `network`) ; `RegisterDomain` → TLD du nom enregistré
   (`UnknownTld` sinon, D1/M7c) **et ouvert** (`TldClosed` sinon, M8b)
   puis nom libre ; `UpdateDomain` → domaine connu, séquence exacte
   `current + 1`, propriétaire ; `RegisterTld` → TLD libre ;
   `TransferTld`/`RevokeTld`/`SetTldOpen` → TLD connu + signataire
   propriétaire ; `AssignDomain` → propriétaire du TLD + nom libre ;
   `RenewTld` → TLD privé connu + propriétaire, terme fixe re-dérivé,
   pré-contrôle de fenêtre de grâce ; `RenewDomain` → domaine connu +
   propriétaire, pré-contrôle PoW de renouvellement (liée à
   l'expiration courante) à la hauteur suivante ;
4. insertion au mempool (capacité fixe, dédup par `TxId` —
   re-soumettre une tx déjà dans le pool est un no-op) ;
5. diffusion aux pairs (sauf l'émetteur).

### Réception d'un bloc (P2P, sync, ou auto-produit)

1. `push_block` : validation complète existante (parent = sommet
   canonique, hauteur, racine de Merkle recalculée, chaque transaction
   validée puis appliquée — atomique par bloc) ;
2. si accepté : le relais lit le résultat d'application
   (`push_block_with_gc` → hachage + domaines retirés par le GC
   déterministe) et persiste via `store_block_with_removals`
   (ajout delta atomique : bloc + sommet + états modifiés — upserts ET
   retraits : domaines expirés, TLDs révoqués). Un redémarrage ne peut
   donc jamais ressusciter un enregistrement expiré ;
3. réconciliation du mempool (les tx incluses sont retirées) ;
4. diffusion aux pairs (sauf l'émetteur).

Un bloc invalide est rejeté avec une erreur typée ; le relais continue
de tourner. Pas de panic sur des données réseau. Les applications
lourdes (Argon2 + commit redb) passent par `tokio::task::block_in_place`
(pilote sync, accept p2p, production) : le runtime déplace les tâches
prêtes vers d'autres workers — appliquer un bloc ne bloque plus le
service DNS ni les métriques.

### Snapshot DNS (publication)

Après CHAQUE bloc appliqué (accept p2p — réorgs incluses —, pilote
sync, production) ET au démarrage (un redémarrage ne doit pas
NXDOMAINer des noms vivants jusqu'au prochain bloc), le relais publie
le **snapshot DNS faisant autorité** sur le flux `watch` lu par la
surface DNS (spécification complète dans `dns.md`) :

- extension du sommet (le chemin écrasant) : **delta O(touched)**,
  copy-on-write — les transactions du bloc nomment les domaines
  touchés, plus les retraits du GC déterministe
  (`AppliedBlock.gc_removed_domains`, évalué au push) ;
- réorg / démarrage : **reconstruction complète** (une passe paginée
  de `ChainState::dns_facts`).

### Production de blocs (M5 : producteur signé)

Toutes les `produce_interval` secondes (2 s par défaut) : si le mempool
est non vide, un bloc est assemblé par `BlockBuilder`
(`timestamp = now`) **signé par un producteur autorisé** (charge
`SCONE-BLOCK-V1` dans `header.consensus`, voir
`/docs/en/technical/blockchain.md`) et suit exactement le chemin
d'acceptance d'un bloc reçu. Un relais sans tx ne produit rien.

Clé de production : la clé d'ancre (`--anchor-key`) quand le relais en
est armé — une ancre est un producteur autorisé — sinon la clé devnet
fixe, valide tant que le pool de producteurs est vide (amorçage ouvert :
genèse). Dès qu'un domaine/TLD vivant existe, un bloc signé par la clé
devnet est rejeté (`InvalidProducer`).

### Boucle d'ancrage (finalité, M5)

Avec `--anchor-key` (fichier de clés `.sconekey`, phrase de passe via
`--anchor-passphrase-env`, défaut `SCONE_ANCHOR_PASS` — le défaut de
la bibliothèque `scone-network` `SCONE_ANCHOR_PASSPHRASE` n'est jamais
atteint via le binaire, clap le surcharge toujours) :

- **rotation du proposant** : exactement UN membre désigné du comité
  auto-propose par époque (`comité[époque % taille]`, déterministe) ;
  les autres membres ne signent jamais leur propre sommet — le
  contenu d'un point de contrôle dépend du sommet du proposant, et
  des auto-propositions non filtrées auraient fragmenté l'époque en
  contenus rivaux que chaque garde refuse à raison comme équivoque
  (testnet, 2026-10-21) ;
- après chaque bloc accepté/produit, si ce nœud est membre du comité
  courant **et** que le comité atteint
  `MIN_FINALITY_COMMITTEE_SIZE`, le relais signe le point de contrôle
  proposé via son `SignerGuard` (`signer-state.dat` dans le
  data-dir — une clé = un vote/époque, fail-safe anti-crash) et le
  propage en gossip ;
- **vote à réception** : une proposition dont le contenu est valide
  sur notre chaîne (le bloc référencé y est, au même haché) et
  portée par une signature de membre du comité reçoit NOTRE
  signature aussi — sans elle, chaque ancre ne signait que le
  contenu de son propre sommet et aucun quorum ne s'assemblait ;
- les signatures reçues (`Message::Checkpoint`) sont accumulées par
  hachage `CheckpointData` (bornées à 256 entrées) ; à
  `quorum_for(taille du comité)` signatures de membres sur les MÊMES
  données → `accept_checkpoint` + diffusion du point de contrôle
  finalisé ;
- **persistance de la finalité** : la fenêtre de checkpoints et la
  base gelée sont écrites dans le store à chaque finalisation et
  rechargées au démarrage. Les checkpoints sont de l'état de gossip
  (jamais portés par les blocs) — un redémarrage sans cette
  persistance repartait d'une base vide et ne pouvait plus jamais
  enchainer une fenêtre servie par les pairs (les ancres d'époque
  sous la fenêtre du réseau ont disparu) ;
- **rattrapage de finalité** : un relais dont la base est vide (boot
  d'un nouveau nœud) demande `GetCheckpoints` à un pair à chaque
  tick tant que la base n'est pas restaurée ;
- un relais sans clé d'ancre reste **passif** : il accumule, finalise
  et relaie, mais ne signe jamais ;
- `GetCheckpoints` → la fenêtre `checkpoint_window()` est servie ;
- toute erreur de la boucle est journalisée et avalée — jamais fatale ;
- le statut (`GetStatus`) expose `checkpoints` et `finalized_epoch`.

### Parking de forks et réorgs (correctif minorité-collante)

Un bloc sœur à la même hauteur, ou un bloc dont le parent n'est pas le
sommet courant (`ParentNotTip`/`UnknownParent`), n'est JAMAIS perdu :
il est **parqué** (map bornée, `PARKED_CAP = 2048` blocs, ~KiB
chacun — RAM bornée au pire cas ; un parking plein ignore
silencieusement les nouveaux venus). Les hachages déjà proposés à
`accept_block` sont dédupliqués (`seen_blocks`) : un bloc perdant
re-gossipé n'est pas ré-évalué.

À CHAQUE tick `Produce`, `reconsider_parked` :

1. vidange linéaire des blocs parqués dont le parent est devenu le
   sommet ;
2. si une branche parquée est plus forte que le sommet courant (règles
   de fork-choice INCHANGÉES : plancher de finalité, chaîne la plus
   longue, hachage le plus bas), adoption — le suffixe déplacé est
   lui-même parqué (les DEUX côtés du fork restent évaluables).

Ceci décolle un nœud qui avait adopté le côté perdant d'un fork à
hauteurs égales : le bloc gagnant, proposé une fois puis refusé, est
périodiquement ré-évalué au lieu d'être oublié pour toujours (constat
chaos-30 : deux blocs sœurs à 15 ms d'écart).

**Persistance des réorgs** : adopter une autre branche ne peut pas
passer par l'ajout monotone (`NonMonotonicHeight`). Le relais applique
`rewind_to_height(point de fork)` puis `persist_state_snapshot`
(réécriture de l'état vivant depuis l'export de la chaîne) puis un
ré-ajout monotone du suffixe gagnant (voir `storage.md`). Un échec de
persistance est journalisé et dégradé (guérison au démarrage par
replay) — jamais fatal.

## Sync (rattrapage)

À chaque connexion à un pair, le relais demande
`GetBlocks(start = hauteur_locale + 1)`. Chaque `Block` reçu qui
complète exactement la hauteur attendue déclenche la requête de la
hauteur suivante, jusqu'à épuisement (`Pong` = rien de plus) ou
`MAX_SYNC_ROUNDS` (10 000) lots — une borne absolue même contre un pair
menteur. Les blocs reçus passent par la validation complète : un pair
malveillant ne peut pas faire accepter quoi que ce soit d'invalide.

### DHT d'enregistrements

- **put** (flux API/web — le canal hex de la CLI a été supprimé) :
  décodage strict borné (`MAX_DHT_CACHE_ENTRY`), vérification contre
  l'état on-chain (propriétaire, séquence, `record_hash` recalculé —
  voir « Vérification des enregistrements »), puis `kad.put_record`
  (stockage local + réplication vers les pairs les plus proches,
  `Quorum::Majority` du facteur de réplication) et cache redb
  persistant.
- **get** (`domain resolve`) : `kad.get_record` ; l'attente est liée à
  son `QueryId` et au `DomainId` demandé — une réponse ne résout que
  la requête correspondante, et la clé de l'enregistrement doit être
  exactement le domaine demandé (vérifié **avant** la vérification
  on-chain). Le résultat est ensuite vérifié contre la chaîne avant
  d'être retourné. Cap de requêtes simultanées : `MAX_DHT_WAITERS`
  (256) ; au-delà, erreur immédiate. Une résolution vérifiée avec
  succès est écrite dans le cache persistant (M5) : les lectures
  locales (`domain_info`) la servent ensuite sans nouvelle requête
  DHT.
- Kademlia tourne en mode serveur, sans TTL d'enregistrement (la
  chaîne gouverne la validité, pas le temps).

### Vérification des enregistrements (règle « la chaîne fait autorité »)

Un `SignedDnsRecord` résolu ou reçu n'est accepté/retourné que si :

1. le domaine est enregistré on-chain ;
2. `record.owner` = propriétaire on-chain courant ;
3. `record.sequence` = séquence on-chain courante ;
4. `record_hash(record.record)` (BLAKE3 recalculé) = le
   `record_hash` engagé on-chain.

Sinon : rejet silencieux côté P2P, erreur explicite côté CLI. La
signature Ed25519 de l'enregistrement est **vérifiée à la réception**
par le relais (`ResolveLocal` : la clé publique du propriétaire est
connue on-chain, la signature doit déverrouiller — un enregistrement
signé par une autre clé est rejeté, `record rejected`) ; elle est aussi
transportée pour les vérificateurs futurs — l'engagement on-chain reste
le contrôle d'intégrité primaire.

## Métriques Prometheus

Un point d'accès HTTP (format texte Prometheus 0.0.4, exposition écrite
à la main — pas de nouvelle dépendance) servi par une tâche dédiée :

- **Désactivé par défaut** : aucun serveur HTTP ne tourne tant que
  `--metrics-addr` n'est pas fourni (les métriques sont opt-in).
- `--metrics-addr ADDR:PORT` : lie le point d'accès sur cette adresse
  (ex. `127.0.0.1:9101` en local, `0.0.0.0:9101` pour l'exposer).
- `--metrics-token TOKEN` : bascule en mode **privé** —
  `Authorization: Bearer *** required (comparaison à temps
  constant, `401` sinon ; le jeton n'apparaît jamais dans les logs
  ni dans l'exposition).
- Un échec de bind **désactive le point d'accès** (avertissement non
  fatal) — jamais le nœud.

Le bind effectif est annoncé sur stdout (contrat lisible par machine,
comme `p2p:`) : une ligne `metrics: <addr>` dès l'écoute.

**19 compteurs** (`counter`, monotones — chemins chauds :
`fetch_add(Relaxed)` sur un `Arc<Metrics>` partagé, jamais une entrée
de consensus) :

| Groupe | Séries |
|---|---|
| Chaîne | `scone_blocks_applied_total`, `scone_blocks_rejected_total`, `scone_checkpoints_signed_total`, `scone_checkpoint_sigs_received_total` |
| Mempool | `scone_txs_admitted_total`, `scone_txs_rejected_total`, `scone_txs_evicted_grace_total`, `scone_txs_included_total` |
| DNS | `scone_dns_queries_total`, `scone_dns_cache_hits_total`, `scone_dns_cache_misses_total`, `scone_dns_local_chain_total`, `scone_dns_dht_answers_total`, `scone_dns_rejected_unverified_total` |
| DHT | `scone_dht_pulls_issued_total`, `scone_dht_negative_hits_total`, `scone_dht_negative_expired_total`, `scone_dht_negative_capped_total`, `scone_dht_cache_evictions_total` |
| Réseau de pairs | `scone_peer_connections_established_total`, `scone_peer_connections_closed_total`, `scone_peer_connections_incoming_total`, `scone_peer_dial_failures_total`, `scone_peer_blocks_received_total`, `scone_peer_transactions_received_total` |

**6 jauges** (`scone_height`, `scone_mempool`, `scone_peers`,
`scone_domains`, `scone_finalized_epoch`, `scone_addr_book_entries`)
— les cinq premières rafraîchies toutes les 15 s via le canal
`RelayCommand` (même chemin que le `status` de la CLI — source de
vérité unique) ; `scone_addr_book_entries` est maintenu directement
par le registre (mis à jour à chaque insertion dans l'annuaire, source
de vérité).

Test E2E : `metrics_endpoint_public_and_private` (200 +
exposition, 401 sans jeton/avec un mauvais jeton, 200 avec Bearer).

## Contrôle CLI (unification RPC→p2p)

Le serveur RPC TCP JSON a été **supprimé** : la CLI (et le pont axum
du registrar) parle le protocole p2p **directement** — même wire
(`/scone/reqres/1`), mêmes gestionnaires côté relais, zéro canal
parallèle. Chaque commande ouvre une connexion QUIC vers la multiaddr
du nœud (`--peer`), échange UN `Message`, ferme.

Messages de contrôle servis par le relais :

- `GetStatus` → `NodeStatus { network, nat_reachable, tip,
  height, domain_count, mempool, peers, finalized_epoch,
  checkpoints, fresh_operators }` (affiché par `scone status` —
  `fresh_operators` = opérateurs avec un battement de cœur frais, RFC
  proof-of-service) ;
- `LookupName { name }` → `NameState` (état on-chain, comme
  `lookup`) ;
- `GetOwnedNames { owner }` → `OwnedNames` (domaines + TLDs vivants
  d'un `OwnerId`) ;
- `Transaction (tx)` → soumission avec gossip gratuit (le relais
  rediffuse aux pairs) ;
- `GetRecord { domain_id }` / `Record (r)` — résolution et publication
  DHT (comme `scone domain resolve` côté CLI) ;
- rejet applicatif → `Message::Error(msg)` (atteint le client comme
  message lisible, pas un EOF).

Le client monte un swarm libp2p minimal (request-response + identify),
timeout d'appel 30 s (aller-retour 20 s). Défaut `--peer` :
`/ip4/127.0.0.1/udp/4001/quic-v1/p2p/12D3KooW…` (nœud de dev local ;
la multiaddr complète avec le vrai peer id est annoncée par le relais
sur stdout via la ligne `p2p:`).

Le canal interne `RelayCommand` survit pour la surface DNS embarquée
(`ResolveLocal`, un aller-retour en processus, jamais d'attente DHT —
voir `dns.md`).

## CLI

```
scone relay [--network testnet|mainnet|devnet] [--data-dir DIR] [--listen MULTIADDR] [--bootstrap MULTIADDR]... [--no-builtin-seeds]
            [--mdns] [--upnp] [--rpc ADDR (obsolète, ignoré)]
            [--anchor-key PATH] [--no-anchor-auto] [--anchor-passphrase-env VAR] [--genesis FILE]
            [--geo-config FILE] [--geo-ip-table PATH] [--geo-country U16] [--geo-continent U16] [--geo-asn U32] [--geo-region U32]
            [--health-check] [--no-health-check] [--health-ttl-secs U64] [--health-port U16]
            [--topology-profile tiny|small|default|high|very-high|devtest]
            [--dns ADDR] [--dns-upstream ADDR]... [--dns-cache-limit U] [--dns-cache-max-mem SIZE] [--dns-max-inflight U]
            [--metrics-addr ADDR:PORT] [--metrics-token TOKEN]
scone status [--peer MULTIADDR]
scone domains [--identity ID | --pubkey HEX] [--peer MULTIADDR]
scone domain register NAME [--identity ID] [--dir DIR] [--passphrase-env VAR] [--peer MULTIADDR]
scone domain set NAME TYPE VALUE [TTL] [--identity ID] [--peer MULTIADDR]   # enregistrement unique express
scone domain update NAME --file FILE [--identity ID] [--peer MULTIADDR]
scone domain get NAME [--peer MULTIADDR]
scone domain renew NAME [--identity ID] [--peer MULTIADDR]
scone domain resolve NAME [--peer MULTIADDR]

# --peer vaut par défaut le relais local (la variable SCONE_PEER surcharge) ;
# --identity vaut par défaut l'identité unique du coffre de clés quand il
# n'y en a qu'une seule. Supprimés (les stubs suggèrent les remplacements) : show,
# submit, lookup, owned.
```

Chaque commande de contrôle parle QUIC directement à la multiaddr du
relais (`--peer`, nœud local par défaut — unification RPC→p2p, plus de
canal TCP JSON). Relais absent → message clair « p2p node unreachable —
is a relay running? », sortie non nulle. Le drapeau `--rpc` est
conservé mais **obsolète et ignoré** (le serveur RPC a été supprimé ;
compatibilité de scripts uniquement).

### Configuration par environnement (`.env`)

Au démarrage, `scone` charge (via `dotenvy`, avant le parsing clap) un
fichier `.env` du répertoire courant : fichier manquant = no-op
silencieux, jamais une erreur. Les valeurs chargées deviennent la
**base environnementale** des options qui déclarent une variable
(`env = "…"` dans `cli.rs`) ; **un flag CLI explicite gagne toujours**
sur `.env`, et `.env` gagne sur les défauts intégrés :

```text
flag CLI explicite  >  variable .env  >  défaut intégré
```

Le fichier `.env.example` à la racine du dépôt documente chaque
variable (copier vers `.env` et ajuster ; `.env` est **gitignoré** — ne
jamais committer un vrai `.env`). Variables :

| Variable | Flag miroir | Sémantique |
|---|---|---|
| `SCONE_NETWORK` | `--network` | `testnet` (défaut), `mainnet`, `devnet` |
| `SCONE_DATA_DIR` | `--data-dir` | répertoire de données (défaut `~/.scone/<network>/`) |
| `SCONE_LISTEN` | `--listen` | multiaddr d'écoute P2P (défaut : toutes interfaces, port éphémère) |
| `SCONE_BOOTSTRAP` | `--bootstrap` | seeds de bootstrap, **répétable** (une ligne par valeur — dotenvy accumule) ; remplace les seeds intégrées |
| `SCONE_GENESIS` | `--genesis` | fichier de genèse (nœuds fondateurs) |
| `SCONE_MDNS` | `--mdns` | découverte mDNS LAN (`true` = activée, `false` = coupée ; défaut : ACTIF sur devnet, INACTIF ailleurs) |
| `SCONE_UPNP` | `--upnp` | mappage de port UPnP (`true` = activé) |
| `SCONE_DNS_ADDR` | `--dns` | adresse de bind du serveur DNS UDP |
| `SCONE_DNS_UPSTREAM` | `--dns-upstream` | replis récursifs, **répétable** |
| `SCONE_DNS_CACHE_LIMIT` | `--dns-cache-limit` | plafond d'entrées du cache DNS |
| `SCONE_DNS_CACHE_MAX_MEM` | `--dns-cache-max-mem` | plafond mémoire du cache DNS (p. ex. `64MB`) |
| `SCONE_DNS_MAX_INFLIGHT` | `--dns-max-inflight` | plafond de requêtes DNS concurrentes |
| `SCONE_METRICS_ADDR` | `--metrics-addr` | adresse du point d'accès métriques (`ADDR:PORT`) |
| `SCONE_METRICS_TOKEN` | `--metrics-token` | jeton bearer du point d'accès métriques |
| `SCONE_TOPOLOGY_PROFILE` | `--topology-profile` | profil de topologie (`tiny`/`small`/`default`/`high`/`very-high`/`devtest` ; défaut `default`) |
| `SCONE_ANCHOR_KEY` | `--anchor-key` | fichier de clé d'ancre (point de contrôle + production) |

`SCONE_ANCHOR_PASS` n'est PAS la variable d'une option : c'est le NOM
PAR DÉFAUT de la **variable contenant la phrase de passe d'ancre**
(`--anchor-passphrase-env`, défaut `SCONE_ANCHOR_PASS` au niveau CLI) —
le relais lit la phrase de passe dans la variable que ce flag nomme.

**Ancre auto (défaut ACTIF)** : sans `--anchor-key`, un fichier de clé
d'ancre est généré/persisté automatiquement dans le répertoire de
données (`anchor.sconekey`, phrase de passe `SCONE_ANCHOR_PASS` si
définie, sinon générée aléatoirement et écrite dans `anchor.pass`
(permissions `0600`, échec d'écriture signalé — la passphrase perdue
rend le keyfile irrécupérable)) — le relais devient producteur dès que
sa clé possède des domaines/TLDs vivants (l'éligibilité PoS n'est
jamais gratuite). `--no-anchor-auto` garde le relais strictement
passif. La variable env par défaut de `--anchor-passphrase-env` est
`SCONE_ANCHOR_PASS`.

`--network` : le défaut suit le BUILD (mainnet sur les binaires
activés mainnet, testnet sinon) ; `devnet` est le réseau local
réinitialisable (genèse `scone-devnet` propre, PoW 0..=2 bits quasi
nulle — effacer `~/.scone/devnet/` et relancer réinitialise la chaîne).
Définit la genèse, les difficultés PoW et le data-dir par défaut
(`~/.scone/<network>/`) ; une tx d'un autre réseau est refusée de façon
typée (`WrongNetwork`).

La surface hex hors ligne (les sous-commandes `tx` et `submit`) a
disparu (remove-tx-surface-v2). La CLI couvre les chemins utilisateur
(`domain register`, `domain renew`, `domains`) ET la famille M8b
comme commandes de premier rang : `tld open`, `tld transfer`,
`tld revoke`, `domain assign`, `domain transfer`. Les claims
`register` minent leur PoW à la difficulté du réseau (millisecondes
sur devnet).

Les options `--geo-*`/`--health-*` configurent le GeoDNS et la
vérification de santé **du résolveur DNS embarqué** : sémantique
complète, ordre de précédence et table GeoIP locale documentés dans
[`dns.md`](dns.md#geodns-geographic-routing) — le routage géographique
est **toujours actif** (table GeoIP embarquée, aucune option requise) ;
`--health-check` est coupé par défaut, chemin classique inchangé.

### `domain register|update` (M5) — transactions signées en une commande

Ces commandes enchaînent construction → signature (coffre de clés) →
soumission → (confirmation) en une invocation unique.

- **register** : construit un `RegisterDomain` signé (timestamp = now,
  preuve vide en devnet), le soumet, attend la confirmation on-chain
  (séquence ≥ 0, timeout 30 s). Un nom déjà enregistré est refusé côté
  client ET côté relais.
- **update** : le fichier d'enregistrements est la seule source de
  vérité. La séquence (on-chain + 1) et le `record_hash` (BLAKE3
  canonique des enregistrements du fichier) sont **dérivés**, jamais
  saisis : signataire et vérificateurs ne peuvent pas diverger. Après
  la confirmation de l'UpdateDomain, le `SignedDnsRecord`
  (enregistrements du fichier + propriétaire + signature Ed25519 sur
  l'encodage canonique) est publié dans la DHT via `put_record` (qui
  re-vérifie contre la chaîne). Un domaine non enregistré, une identité
  non propriétaire ou une erreur de fichier échouent proprement, sans
  toucher la chaîne.

#### Fichier d'enregistrements

Texte, un enregistrement par ligne, `#` = commentaire, lignes vides
ignorées ; l'encodage canonique trié fait que l'ordre du fichier n'a
aucun effet sur le hachage engagé. Le fichier est l'ensemble COMPLET
(une mise à jour remplace, elle ne fusionne pas). Types :

```text
A 192.0.2.1
AAAA 2001:db8::1
CNAME www.example.uip
NS ns1.example.uip
MX 10 mail.example.uip
TXT texte libre (espaces normalisés)
```

Erreurs typées : type inconnu, IP/nom invalide (avec numéro de ligne),
ensemble vide, doublon, > `MAX_RECORDS_PER_SET`.

## `--genesis` (fondateurs)

`scone relay --genesis genesis.bin` — **nœuds fondateurs uniquement**
(campagne kickstarter ; le fichier est produit par le binaire
`scone-genesis`). Règles normatives :

1. Le fichier est rejeté **avant lecture** s'il dépasse 1 MiB, puis
   strictement décodé (`decode_complete` : aucun octet traînant). Le
   hachage du bloc 0 est **recalculé**, jamais lu depuis le fichier.
2. Validation sémantique : hauteur 0, `consensus` commençant par
   l'identifiant de réseau du réseau demandé (`--network`), aucune
   transaction, manifeste d'allocation canonique. Toute violation =
   erreur typée au démarrage (jamais de panic, le démon ne démarre
   pas).
3. Data-dir **vierge** : la chaîne naît sur la genèse allouée — les
   allocations sont appliquées en RAM **avant toute écriture disque**,
   puis persistées (états initiaux + bloc genèse sous une clé meta). Le
   premier bloc produit ne peut donc qu'étendre la genèse allouée.
4. Data-dir **déjà productif** : la genèse de naissance du store est le
   `prev_hash` du bloc 1 (auto-certifiant). Hachage de fichier égal =
   OK silencieux (l'option est **redondante après le premier
   démarrage**) ; différent = refus typé — un nœud rejoint un réseau
   via sync, jamais via `--genesis`.
5. Redémarrage **sans** l'option : un data-dir né fondateur retrouve sa
   genèse allouée via la clé meta (relien RAM à chaque démarrage).

L'épinglage constant du hachage de genèse (anti divergence de réseau)
est un chantier séparé.

### Manifeste du binaire `scone-genesis` (kickstarter)

Le binaire `scone-genesis` (crate `scone-genesis`) prépare le manifeste
`genesis-manifest.json` (format `ALLOC-V1`) puis produit
`genesis.bin`. Règles normatives :

- `scone-genesis new --network <network>` crée un manifeste vide ;
  `add`/`remove`/`list` gèrent les allocations, `build` produit
  `genesis.bin` (+ hachage), `verify` le revalide.
- `add domain <name>` exige que le **TLD du domaine soit déjà alloué
  dans le manifeste** (rejet immédiat sinon — la chaîne l'aurait
  refusé à l'application de la genèse) ; son `valid_until` est
  **automatique** : `born_at + 1 an` (le default_validity du
  manifeste), jamais saisi à la main.
- `add tld <name>` : **ouvert par défaut** (à personne,
  libre-service, plancher 1 an, propriétaire = zéro sur le wire —
  genre `0x03`) ; `--private` = privé (propriétaire, terme genèse
  `born_at + TLD_TERM_SECS` — 3 ans, cycle de vie standard :
  `RenewTld`, grâce de 60 jours).
- Tri canonique par nom (contenus égaux → même bloc → même hachage,
  quel que soit l'ordre d'insertion) ; doublons refusés.

## Limites et garanties mémoire

| Ressource | Bornes |
|---|---|
| Trame P2P | `MAX_MESSAGE_LEN` (1 MiB) avant parsing |
| Mempool | 4096 tx (rejet typé au-delà) ; `MAX_PENDING_PER_DOMAIN = 4` tx en attente par domaine |
| Grâce mempool | `MAX_GRACE_ATTEMPTS = 10` réinsertions d'une tx évacuée pour un motif non définitif (≈ 7 min à 40 s/cycle), puis éviction |
| Parking de forks | `PARKED_CAP = 2048` blocs concurrents |
| Points de contrôle accumulés | `MAX_PENDING_CHECKPOINTS = 256` (éviction : plus petit hachage de données) |
| Attentes DHT simultanées | `MAX_DHT_WAITERS` (256) ; recherche DHT bornée à 30 s (`DHT_LOOKUP_TIMEOUT`) |
| Blocs par réponse de sync | 1 par aller-retour ; ≤ 10 000 lots par pair (`MAX_SYNC_ROUNDS`) ; borne protocole `MAX_BLOCKS_PER_REQUEST = 128` sur la requête |
| Enregistrement DHT | `MAX_DHT_CACHE_ENTRY` (64 KiB) |
| Cache DHT persistant (disque) | double borne LRU : `DHT_CACHE_MAX_ENTRIES` (100 000 entrées) ET `DHT_CACHE_MAX_BYTES` (2 GiB) — la première atteinte évince par ordre de clé ; éviction GC dans la MÊME transaction que le bloc (un domaine tué par le GC de la chaîne ne laisse jamais d'entrée orpheline) |
| Cache négatif DHT | `DHT_NEGATIVE_TTL_SECS` (30 s), cap FIFO `DHT_NEGATIVE_CAP` (4 096) |
| Chaîne en RAM | genèse + fenêtre de sommet (historique servi depuis le store) |

Règle anti-boucle (H2) : une transaction n'est re-diffusée que si elle
a été **réellement insérée** dans le mempool (première vue) ; un
doublon (déjà dans le pool ou déjà dans la chaîne) n'est jamais
re-relayé. Un bloc déjà canonique est détecté et ignoré avant
`push_block` (pas de re-stockage, pas de re-diffusion).

Décisions ouvertes : identité réseau persistante du relais, gossip de
diffusion (actuellement : request-response unicast vers chaque pair
connu). Résolus depuis la rédaction initiale : vrai fork choice
(parking + reconsider_parked, voir ci-dessus), consensus PoW
(producteur signé + comité d'ancres), résolveur DNS (la surface
`--dns`, voir dns.md), canal de contrôle (unification RPC→p2p : plus
de TCP JSON).

## Arrêt propre (SIGTERM / SIGINT)

Le démon `scone relay` (premier plan) capte SIGTERM et SIGINT via
`tokio::signal` : au signal, le futur `Relay::run` est annulé
(`tokio::select!` dans le binaire), ce qui droppe le Swarm libp2p et
ferme les sockets QUIC. Le runtime tokio est ensuite arrêté avec un
arrêt borné (≤ 2 s) qui droppe les tâches DNS détachées (UDP/TCP) et
ferme leurs sockets — l'objectif est qu'un `kill` ne laisse aucune
socket UDP orpheline (cas observé sur le testnet 60). Un arrêt piloté
par signal journalise `relay shutting down (signal)` (INFO, stderr) et
sort avec un code de sortie **0** : un arrêt demandé n'est pas un
échec. Si l'installation des gestionnaires a échoué, le comportement
par défaut du signal s'applique (dégradation, jamais de panic).


## Transport Tor (question ouverte)

QUIC/UDP ne traverse pas les proxys SOCKS simples, donc les relais ne
peuvent pas tourner trivialement sur Tor aujourd'hui. Un transport Tor
natif libp2p (relais à adresse onion) est le chemin suivi — voir le
guide de résistance à la censure du site web pour l'état actuel et le
pont CNAME `.tor` (noms lisibles pour les services cachés, pris en
charge aujourd'hui).

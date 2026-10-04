# Relay réseau (M4) — architecture, flux et contrôle du CLI

> Spécification normative du relais réseau Scone (`scone-network`).
> Toute divergence doc/code = bug (corriger la source de vérité).

## Vue d'ensemble

Le **relay** est le nœud P2P complet de Scone : il maintient la
blockchain canonique (RAM + `NodeStore` redb), relaie transactions et
blocs, produit des blocs (mode devnet), sert le protocole de sync,
publie et résout les `SignedDnsRecord` dans une DHT Kademlia, et
expose le **protocole p2p** (`/scone/reqres/1`) que le CLI
interroge directement (unification RPC→p2p : le canal TCP JSON a
disparu).

```text
                    ┌────────────────────────────────────────┐
                    │              scone relay               │
  P2P (libp2p)      │                                        │
  QUIC (udp,        │  Swarm libp2p                          │
  port éphémère     │  ├─ identify  (découverte d'adresses)  │
  par défaut) ◄─────┤  ├─ ping      (liveness)               │
                    │  ├─ kad       (DHT des records)        │
                    │  └─ reqres    (messages scone-prot.)   │
                    │                                        │
                    │  Relay core (une seule tâche)          │
                    │  ├─ Blockchain (validation totale)     │
                    │  ├─ Mempool (borné, dédup par TxId)    │
                    │  ├─ BlockBuilder (production devnet)   │
                    │  ├─ Parking fork (borné, 2048)         │
                    │  └─ Sync driver (rattrapage par lots)  │
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

Bornes matérielles au niveau du swarm (comportement libp2p
`connection_limits`, la première entrée de `SconeBehaviour`) :

- `MAX_TOTAL_CONNECTIONS = 128` — plafond de connexions établies
  (entrantes + sortantes) par relay ; l'excédent entrant est REFUSÉ,
  jamais mis en file ;
- `MAX_PER_PEER_CONNECTIONS = 4` — un seul pair ne peut pas monopoliser
  le quota total en ouvrant beaucoup de connexions ;
- `MAX_PENDING_INCOMING = 32` — garde slowloris : les demi-connexions
  non authentifiées n'attendent jamais indéfiniment.

Ces bornes complètent les limites existantes par message
(`MAX_MESSAGE_LEN` 1 Mio) et par budget (sync `max ≤ 128` blocs, DNS
TCP 64 connexions/512 o/5 s). Un attaquant par inondation de
connexions obtient des refus, pas une croissance mémoire.

## Découverte des pairs

Trois mécanismes complémentaires, dans cet ordre de priorité :

1. **Graines explicites** (`--bootstrap`, multiaddrs complets) —
   toujours prioritaires ; fournir une liste non vide REMPLACE tout le
   reste. `--no-builtin-seeds` vide les graines embarquées pour les
   déploiements 100 % privés.
2. **Graines embarquées** (`builtin_seeds(network)` dans
   `scone-network/src/config.rs`) — figées à chaque release officielle
   du binaire, par réseau. Un relay sans configuration rejoint le
   réseau seul. (Vide en testnet tant que la testnet publique n'est
   pas déployée.)
3. **Annuaire appris** (`<data-dir>/peers.json`) — chaque adresse
   apprise via `identify` est retenue (dernière vue, plafond 200
   entrées) et sauvegardée ~1×/min au tick de production. Au boot sans
   graine ni bootstrap, l'annuaire est recomposé : le relay retrouve
   ses pairs habituels sans configuration. C'est un cache : fichier
   absent/corrompu = annuaire vide, jamais une erreur.

### Optimisation locale de topologie (`--topology`, opt-in)

Le relay peut optimiser activement son voisinage à l'aide du moteur
pur `scone-topology` (voir la doc de la crate). **Désactivé par
défaut** ; `--topology` l'arme, les limites sont modulables :

```
scone relay --topology \
  [--topology-min-connections 8] \
  [--topology-target-connections 16] \
  [--topology-max-connections 32] \
  [--topology-optimization-interval-secs 60] \
  [--topology-min-score-improvement 0.15]
```

Une config incohérente (`min > target > max`, seuil hors `[0,1]`
— `NaN` compris —, intervalle nul) est une **erreur de démarrage
typée**, jamais un comportement silencieux. Des flags `--topology-*`
fournis sans `--topology` déclenchent un avertissement (ils sont
ignorés).

**Vue (`TopologyView`)** — alimentée par les événements du swarm :

- `ping` réussi → observation RTT (fenêtre médiane 8 échantillons) ;
- `ping` échoué → pénalité de fiabilité ;
- `identify` reçu → adresses (≤ 4/pair) + protocoles annoncés
  (bitmask kad/reqres/identify/relay) ;
- connexion établie → état connecté, sens (entrante/sortante),
  ancienneté, circuit-relay détecté ;
- connexion fermée → pénalité + compteur de churn SEULEMENT si non
  sollicitée (une éviction volontaire ne compte pas) ;
- échec de dial sortant → pénalité de fiabilité.

La vue est bornée (500 pairs ; au-delà, seuls les pairs connectés
sont mémorisés).

**`kad_required` depuis les kbuckets** — avant chaque évaluation, le
relay parcourt la table de routage Kademlia : un pair est protégé
s'il est `Connected` dans un bucket, ou `Disconnected` dans un bucket
SATURÉ (k = 20). Un pair protégé n'est JAMAIS choisi comme évincé.

**Boucle d'optimisation** — toutes les
`--topology-optimization-interval-secs` (60 s par défaut) :

1. recalcul de `kad_required` (kbuckets) ;
2. évaluation du moteur (hystérésis : grâce 120 s, amélioration
   minimale de score +15 %, cooldown d'éviction 600 s) ;
3. décision appliquée : `AddCandidate` → dial du candidat ;
   `Replace` → fermeture de l'évincé (UNE fois — progressive, pas de
   boucle) puis dial du remplaçant ;
4. instantané **métriques** loggé (niveau INFO) : comptes
   connus/connectés/entrants/sortants/circuits/protégés, RTT
   moyen/médian, buckets IP distincts, concentration max, churn
   (remplacements/déconnexions).

Le score de voisinage n'est JAMAIS une mesure de confiance (voir la
doc de `scone-topology`) : aucune décision de consensus n'en découle.

### Joignabilité (AutoNAT) et relais de transport (circuit-relay v2)

Chaque relay exécute AutoNAT (client + serveur) : le nœud demande à
ses pairs de tenter un dial RETOUR vers lui ; le verdict
(`reachable` : oui = joignable de l'extérieur, non = derrière NAT,
inconnu = pas encore testé) est rafraîchi toutes les 5 min et exposé
dans `NodeStatus` (champ `nat_reachable`, wire varint 0/1/2) — le CLI
l'affiche (`scone status`). Les adresses privées comptent
(`only_global_ips = false`) : un mesh LAN teste sa joignabilité LAN.

Un relay joignable active le **hop** circuit-relay v2 (il peut
relayer le trafic des pairs NATés) et **DCUtR** (hole-punching : deux
nœuds reliés par circuit tentent une connexion directe et libèrent le
relais). Un relay jugé NATé demande automatiquement une réservation au
premier pair joignable connu et écoute à travers lui (adresse
`/p2p/<relay>/p2p-circuit`).

Règle de production : un nœud NATé relayé ne devrait pas devenir
producteur de blocs (latence/disponibilité dégradées) — la règle
PoS (siège = domaine vivant) ne l'interdit pas encore ; à documenter
comme exigence d'exploitation avant la testnet publique.

## Transport et protocoles libp2p

| Protocole | Nom | Rôle |
|---|---|---|
| QUIC (quic-v1) | — | Transport principal (udp) |
| identify | `/scone/id/1` | Découverte des adresses des pairs (remplit la table Kademlia) |
| ping | libp2p standard | Liveness |
| Kademlia | `/scone/kad/1` | DHT : clé = `DomainId` (32 o), valeur = encodage canonique du `SignedDnsRecord` |
| request-response | `/scone/reqres/1` | Messages applicatifs scone-protocol |

### Cadre applicatif (`/scone/reqres/1`)

Chaque trame est `len: u32 BE || octets` où les octets sont l'encodage
canonique d'un `Message` de `scone-protocol` (le codec existant —
aucun second format). La longueur est bornée par
`limits::MAX_MESSAGE_LEN` **avant** toute lecture ou allocation : un
pair annonçant une trame plus grosse est rejeté sans coût mémoire.

Messages échangés (un par aller-retour request/response) :

| Message | Direction | Réponse |
|---|---|---|
| `Hello { version }` | handshake | `Hello` (version locale) |
| `Ping` / `Pong` | liveness | `Pong` |
| `GetBlock { hash }` | demande | `Block` (fenêtre RAM) ou `Pong` si inconnu |
| `GetBlocks { start, max }` | sync | **un** `Block` (hauteur `start`), ou `Pong` si rien |
| `Block (b)` | relais de bloc | `Pong` (ack) |
| `Transaction (tx)` | relais de tx | `Pong` (ack) |
| `GetRecord { domain_id }` | DHT locale | `Record` (cache local) ou `Pong` |
| `Record (r)` | publication | `Pong` si accepté, erreur sinon |

Règle : **une seule réponse par requête** ; les messages d'acquittement
sont `Pong`. Les réponses de sync portent un bloc à la fois — le
rattrapage itère côté demandeur (voir « Sync »). C'est volontairement
borné : jamais plus d'un bloc en vol par pair.

## Comportement du relay

### Démarrage

0. résolution du réseau (`--network testnet|mainnet|devnet`, défaut
   testnet tant que le projet est en dev — M8b). Le data-dir est **par
   réseau** (`~/.scone/<réseau>/`) : un store non vide qui tient une
   chaîne d'un autre réseau est refusé (`Corrupted`) ;
1. ouverture/creation du store redb (`<data-dir>/chain.redb`) ;
2. `load_chain(store, réseau)` : tip en O(1) + états de domaines
   paginés (fenêtre RAM : genesis + tip) ;
3. construction du swarm (identité Ed25519 éphémère par session —
   la persistance de l'identité réseau est une décision ouverte) ;
4. composition de la liste de démarrage (voir
   [Découverte des pairs](#découverte-des-pairs)) : `--bootstrap`
   explicite (doivent finir par `/p2p/<peer-id>`) SINON graines
   embarquées du réseau SINON annuaire appris (`peers.json`) ;
5. dial de chaque multiaddr et enregistrement dans la table Kademlia
   (un dial mort est ignoré — les autres composent le réseau).

### Réception d'une transaction (P2P ou soumission CLI `submit tx`)

1. décodage strict borné (`MAX_MESSAGE_LEN`) ;
2. validation cryptographique complète (`validate_transaction` :
   binding owner/clé, `verify_strict` sur le payload canonique) ;
3. **anti-replay D'ABORD** (peu coûteux, indépendant de l'état,
   définitif) puis pré-contrôle d'état (miroir bon marché des
   règles `push_block`, le PoW — coûteux — n'est vérifié qu'au
   push) : une `RenewDomain` rejouée répond `TxReplay`, pas une
   erreur PoW mensongère (sa preuve est liée à l'échéance que la
   première application a déplacée). La séparation de
   réseaux est assurée à l'étape 2 : le payload de signature injecte
   le réseau DU RELAY, une tx signée ailleurs échoue en
   `InvalidSignature` (wire 2026-09 : plus de champ `network`) ;
   `RegisterDomain` → TLD du nom enregistré
   (`UnknownTld` sinon, D1/M7c) **et ouvert** (`TldClosed` sinon,
   M8b) puis nom libre ; `UpdateDomain` → domaine connu, séquence
   exacte `current + 1`, propriétaire ; `RegisterTld` → TLD libre ;
   `TransferTld`/`RevokeTld`/`SetTldOpen` → TLD connu + signataire
   owner ; `AssignDomain` → owner du TLD + nom libre ;
   `RenewDomain` → domaine connu + propriétaire, précheck du PoW
   renouvellement (lié à l'échéance courante) à la hauteur suivante ;
4. insertion au mempool (capacité fixe, dédup par `TxId` —
   re-soumettre une tx déjà en pool est un no-op) ;
5. broadcast aux pairs (sauf l'émetteur).

### Réception d'un bloc (P2P, sync ou auto-produit)

1. `push_block` : validation totale existante (parent = tip canonique,
   hauteur, Merkle recalculée, chaque transaction validée puis
   appliquée — atomique par bloc) ;
2. si accepté : le relay lit l'issue d'application
   (`push_block_with_gc` → hash + domaines retirés par le GC
   déterministe) et persiste via `store_block_with_removals` (append
   delta atomique : bloc + tip + états modifiés — upserts ET
   retraits : domaines expirés, TLDs révoqués). Un redémarrage ne
   peut donc jamais ressusciter un enregistrement expiré ;
3. réconciliation du mempool (les tx incluses sont retirées) ;
4. broadcast aux pairs (sauf l'émetteur).

Un bloc invalide est rejeté avec une erreur typée ; le relay continue
de tourner. Aucun panic sur données réseau. Les apply lourds
(Argon2 + commit redb) passent par `tokio::task::block_in_place`
(sync driver, accept p2p, production) : le runtime déplace les
tâches prêtes vers d'autres workers — l'application d'un bloc ne
stoppe plus le service DNS ni les métriques.

### Snapshot DNS (publication)

Après CHAQUE bloc appliqué (accept p2p — reorgs comprises —, sync
driver, production) ET au boot (un redémarrage ne doit pas
NXDOMAINer des noms vivants jusqu'au bloc suivant), le relay
publie le **snapshot DNS autoritaire** sur le flux `watch` lu par
la surface DNS (spécification complète dans `dns.md`) :

- extension de tip (le chemin écrasant) : **delta O(touchés)**
  copy-on-write — les transactions du bloc nomment les domaines
  touchés, plus les retraits du GC déterministe
  (`AppliedBlock.gc_removed_domains`, évalués au push) ;
- reorg / boot : **reconstruction complète** (walk paginé
  `ChainState::dns_facts`, une fois).

### Production de blocs (M5 : producteur signé)

Toutes les `produce_interval` secondes (2 s par défaut) : si le
mempool est non-vide, un bloc est assemblé par `BlockBuilder`
(`timestamp = now`) **signé par un producteur autorisé** (payload
`SCONE-BLOCK-V2` dans `header.consensus`, voir
`/docs/fr/technical/blockchain.md`) et suit exactement le chemin
d'acceptation d'un bloc reçu. Un relay sans tx ne produit rien.

Clé de production : la clé anchor (`--anchor-key`) quand le relay en
est armé — un anchor est un producteur autorisé — sinon la clé devnet
fixe, valide tant que le pool de producteurs est vide (bootstrap
ouvert : genèse). Dès qu'un domaine/TLD vivant existe, un bloc signé
par la clé devnet est rejeté (`InvalidProducer`).

### Boucle anchor (finalité, M5)

Avec `--anchor-key` (keyfile `.sconekey`, passphrase via
`--anchor-passphrase-env`, défaut `SCONE_ANCHOR_PASSPHRASE`) :

- après chaque bloc accepté/produit, si ce nœud est membre du comité
  courant **et** que le comité atteint `MIN_FINALITY_COMMITTEE_SIZE`,
  le relay signe le checkpoint proposé via son `SignerGuard`
  (`signer-state.dat` dans le data-dir — une clé = un vote/epoch,
  crash-safe fail-closed) et le propage (gossip) ;
- les signatures reçues (`Message::Checkpoint`) sont accumulées par
  hash de `CheckpointData` (borné à 256 entrées) ; à
  `quorum_for(taille du comité)` signatures de membres sur le MÊME
  data → `accept_checkpoint` + diffusion du checkpoint finalisé ;
- un relay sans clé anchor reste **passif** : il accumule, finalise
  et relaie, mais ne signe jamais ;
- `GetCheckpoints` → la fenêtre `checkpoint_window()` est diffusée ;
- toute erreur de boucle est journalisée et avalée — jamais fatale ;
- le status (`GetStatus`) expose `checkpoints` et `finalized_epoch`.

### Fork parking et reorgs (fix de la minorité collante)

Un bloc sœur à même hauteur, ou un bloc dont le parent n'est pas
le tip courant (`ParentNotTip`/`UnknownParent`), n'est JAMAIS
perdu : il est **parqué** (map bornée `PARKED_CAP = 2048` blocs,
~KiB chacun — pire cas RAM borné ; un parking plein ignore
silencieusement les nouveaux arrivants). Les hashs déjà offerts à
`accept_block` sont dédupliqués (`seen_blocks`) : un bloc perdant
re-gossipé n'est pas réévalué.

À CHAQUE tick `Produce`, `reconsider_parked` :

1. drain linéaire des blocs parkés dont le parent est devenu tip ;
2. si une branche parkée est plus forte que le tip courant
   (règles fork-choice INCHANGÉES : plancher de finalité, chaîne
   la plus longue, hash le plus bas), adoption — le suffixe
   déplacé est lui-même parqué (les DEUX côtés du fork restent
   évaluables) ;

C'est ce qui débloque un nœud qui avait adopté le côté perdant
d'un fork à égalité : le bloc gagnant, arrivé une fois puis
refusé, est réévalué périodiquement au lieu d'être oublié pour
toujours (finding chaos-30 : deux sœurs à 15 ms d'écart).

**Persistance du reorg** : l'adoption d'une autre branche ne peut
pas passer par l'append monotone (`NonMonotonicHeight`). Le relay
applique `rewind_to_height(point de fourche)` puis
`persist_state_snapshot` (réécriture de l'état vivant depuis
l'export de la chaîne) puis ré-append monotone du suffixe gagnant
(voir `storage.md`). Une défaillance de persistance est
journalisée et dégradée (guérison au boot par rejeu) — jamais
fatale.

## Sync (rattrapage)

À chaque connexion à un pair, le relay demande
`GetBlocks(start = height_locale + 1)`. Chaque `Block` reçu qui
complète exactement la hauteur attendue déclenche la demande de la
hauteur suivante, jusqu'à épuisement (`Pong` = rien de plus) ou
`MAX_SYNC_ROUNDS` (10 000) lots — borne absolue même contre un pair
menteur. Les blocs reçus passent par la validation complète : un pair
malveillant ne peut faire accepter quoi que ce soit d'invalide.

### DHT des records

- **put** (`record put`) : décodage strict borné
  (`MAX_DHT_CACHE_ENTRY`), vérification contre l'état on-chain
  (propriétaire, séquence, `record_hash` recalculé — voir
  « Vérification des records »), puis `kad.put_record` (stockage
  local + réplication aux pairs les plus proches, `Quorum::Majority`
  du facteur de réplication) et cache persistant redb.
- **get** (`record get`) : `kad.get_record` ; le waiter est lié
  à son `QueryId` et au `DomainId` demandé — une réponse ne résout
  que la requête correspondante, et la clé du record doit être
  exactement le domaine demandé (vérifié **avant** la vérification
  on-chain). Le résultat est ensuite vérifié contre la chaîne avant
  d'être retourné. Plafond de requêtes simultanées :
  `MAX_DHT_WAITERS` (256) ; au-delà, erreur immédiate. Une
  résolution vérifiée avec succès est écrite dans le cache persistant
  (M5) : les lectures locales (`domain_info`) la servent ensuite
  sans nouvelle requête DHT.
- Kademlia fonctionne en mode serveur, sans TTL de records (la chaîne
  gouverne la validité, pas le temps).

### Vérification des records (règle « la chaîne est l'autorité »)

Un `SignedDnsRecord` résolu ou reçu n'est accepté/retourné que si :

1. le domaine est enregistré on-chain ;
2. `record.owner` = propriétaire on-chain actuel ;
3. `record.sequence` = séquence on-chain actuelle ;
4. `record_hash(record.record)` (BLAKE3 recalculé) = le
   `record_hash` engagé on-chain.

Sinon : rejet silencieux côté P2P, erreur explicite côté CLI. La
signature Ed25519 du record est **vérifiée à la réception** par le
relay (`ResolveLocal` : la clé publique du propriétaire est sue
on-chain, la signature doit déverrouiller — un record signé par une
autre clé est rejeté, `record rejected`) ; elle est aussi transportée
pour les vérificateurs futurs — l'engagement on-chain reste le
contrôle d'intégrité primaire.

## Métriques Prometheus

Endpoint HTTP (format texte Prometheus 0.0.4, exposition écrite à
la main — aucune dépendance ajoutée) servi par une tâche dédiée :

- **Public par défaut** sur `127.0.0.1:9101` (le CLI applique ce
  défaut ; la valeur par défaut de la bibliothèque est `None` —
  plusieurs relays dans un même processus n'entrent jamais en
  collision).
- `--metrics-addr ADDR|off` : autre adresse, ou `off` = endpoint
  désactivé.
- `--metrics-token TOKEN` : bascule en mode **privé** —
  `Authorization: Bearer <TOKEN>` exigé (comparaison à temps
  constant, `401` sinon ; le token n'apparaît jamais dans les logs
  ni dans l'exposition).
- Un échec de bind **désactive l'endpoint** (warning non fatal) —
  jamais le nœud.

Le bind effectif est annoncé sur stdout (contrat machine-lisible,
comme `p2p:`) : une ligne `metrics: <addr>` dès l'écoute.

**19 compteurs** (`counter`, monotones — hot path :
`fetch_add(Relaxed)` sur un `Arc<Metrics>` partagé, jamais un
consensus) :

| Groupe | Séries |
|---|---|
| Chaîne | `scone_blocks_applied_total`, `scone_blocks_rejected_total`, `scone_checkpoints_signed_total`, `scone_checkpoint_sigs_received_total` |
| Mempool | `scone_txs_admitted_total`, `scone_txs_rejected_total`, `scone_txs_evicted_grace_total`, `scone_txs_included_total` |
| DNS | `scone_dns_queries_total`, `scone_dns_cache_hits_total`, `scone_dns_cache_misses_total`, `scone_dns_local_chain_total`, `scone_dns_dht_answers_total`, `scone_dns_rejected_unverified_total` |
| DHT | `scone_dht_pulls_issued_total`, `scone_dht_negative_hits_total`, `scone_dht_negative_expired_total`, `scone_dht_negative_capped_total`, `scone_dht_cache_evictions_total` |

**5 gauges** (`scone_height`, `scone_mempool`, `scone_peers`,
`scone_domains`, `scone_finalized_epoch`) rafraîchies toutes les
15 s via le canal `RelayCommand` (même chemin que le `status` CLI —
source de vérité unique).

Test e2e : `metrics_endpoint_public_and_private` (200 + exposition,
401 sans/avec mauvais token, 200 avec Bearer).

## Contrôle du CLI (unification RPC→p2p)

Le serveur RPC TCP JSON a été **supprimé** : le CLI (et le pont
axum du registrar) parlent le protocole p2p **directement** —
même wire (`/scone/reqres/1`), mêmes handlers côté relay, zéro
canal parallèle. Chaque commande ouvre une connexion QUIC vers le
multiaddr du nœud (`--peer`), échange UN `Message` et referme.

Messages de contrôle servis par le relay :

- `GetStatus` → `NodeStatus { network, nat_reachable, tip,
  height, domain_count, mempool, peers, finalized_epoch,
  checkpoints }` (l'affiche `scone status`) ;
- `LookupName { name }` → `NameState` (état on-chain, comme
  `lookup`) ;
- `GetOwnedNames { owner }` → `OwnedNames` (domaines + TLDs
  vivants d'un `OwnerId`) ;
- `Transaction (tx)` → soumission avec gossip gratuit (le relay
  re-diffuse aux pairs) ;
- `GetRecord { domain_id }` / `Record (r)` — résolution et
  publication DHT (comme `record get`/`record put`) ;
- refus applicatif → `Message::Error(msg)` (le rejoint le client
  comme message lisible, pas un EOF).

Le client amène un swarm libp2p minimal (request-response +
identify), timeout d'appel 30 s (aller-retour 20 s). Default
`--peer` : `/ip4/127.0.0.1/udp/4001/quic-v1/p2p/12D3KooW…`
(nœud local de dev ; le multiaddr complet avec le vrai peer id
est annoncé par le relay sur stdout via la ligne `p2p:`).

Le canal `RelayCommand` interne survit pour la surface DNS
embarquée (`ResolveLocal`, un aller-retour in-process, jamais de
waiter DHT — voir `dns.md`).

## CLI

```
scone relay [--network testnet|mainnet|devnet] [--data-dir DIR] [--listen MULTIADDR] [--bootstrap MULTIADDR]... [--rpc ADDR (obsolète, ignoré)]
            [--anchor-key PATH] [--no-anchor-auto] [--anchor-passphrase-env VAR] [--genesis FILE]
            [--geo-dns] [--geo-ip-table PATH] [--geo-country U16] [--geo-continent U16] [--geo-asn U32] [--geo-region U32]
            [--health-check] [--no-health-check] [--health-ttl-secs U64] [--health-port U16]
            [--topology] [--topology-min-connections U] [--topology-target-connections U] [--topology-max-connections U]
            [--topology-optimization-interval-secs U64] [--topology-min-score-improvement F64]
            [--dns ADDR] [--dns-upstream ADDR]... [--dns-cache-limit U] [--dns-cache-max-mem SIZE] [--dns-max-inflight U]
            [--metrics-addr ADDR|off] [--metrics-token TOKEN]
scone status [--peer MULTIADDR]
scone submit tx --hex HEX [--peer MULTIADDR]
scone lookup NAME [--peer MULTIADDR]
scone domain register NAME --identity ID [--dir DIR] [--passphrase-env VAR] [--peer MULTIADDR]
scone domain update NAME --file FILE --identity ID [--dir DIR] [--passphrase-env VAR] [--peer MULTIADDR]
scone domain info NAME [--peer MULTIADDR]
scone record put NAME --file FILE [--peer MULTIADDR]      # FILE = hex canonique
scone record get NAME [--peer MULTIADDR]
scone dig NAME [--dns ADDR] [--qtype A|AAAA|TXT|MX|NS|CNAME|ANY]
```

Toutes les commandes de contrôle parlent QUIC direct au multiaddr
du relay (`--peer`, défaut nœud local — unification RPC→p2p, plus
de canal TCP JSON). Relay absent → message clair « p2p node
unreachable — is a relay running? », exit non nul. Le flag
`--rpc` est conservé mais **obsolète et ignoré** (le serveur RPC a
été supprimé ; compatibilité de scripts uniquement).

**Anchor auto (défaut ON)** : sans `--anchor-key`, un keyfile
anchor est généré/persisté automatiquement dans le data-dir
(`anchor.sconekey`, passphrase `SCONE_ANCHOR_PASS` si définie,
sinon auto-générée et écrite dans `anchor.pass`) — le relay
devient producteur dès que sa clé possède des domaines/TLDs
vivants (l'éligibilité PoS n'est jamais gratuite).
`--no-anchor-auto` garde le relay strictement passif. La variable
d'environnement par défaut de `--anchor-passphrase-env` est
`SCONE_ANCHOR_PASS`.

`--network` (M8b) : `testnet` par défaut (projet en dev), `mainnet`
explicite, `devnet` pour le réseau local réinitialisable (genèse
propre `scone-devnet`, PoW quasi nul 0..=2 bits — effacer
`~/.scone/devnet/` et relancer réinitialise la chaîne). Fixe la
genèse, les difficultés PoW et le data-dir par
défaut (`~/.scone/<réseau>/`) ; une tx d'un autre réseau est refusée
de façon typée (`WrongNetwork`).

La surface offline `scone tx build` couvre la famille complète M8b
(`set-tld-open`, `assign-domain`, `renew-domain`, `transfer-tld`,
`revoke-tld`) ; les claims `register`/`register-tld` minent leur PoW
à la difficulté testnet par défaut (quelques ms).

Les options `--geo-*`/`--health-*` arment le GeoDNS et le health
checking **du resolver DNS embarqué** : sémantique complète, ordre de
précédence et table GeoIP locale documentés dans
[`dns.md`](dns.md#geodns-routage-géographique-opt-in) — désactivés
par défaut, chemin classique inchangé.

### `domain register|update` (M5) — transactions signées en une commande

Ces commandes enchaînent build → sign (keystore) → submit →
(confirmation) en une seule invocation ; elles suppriment le
pipe-shell `tx build | tx sign | submit tx` du devnet M4.

- **register** : construit une `RegisterDomain` signée (timestamp = now,
  proof vide en devnet), la soumet, attend la confirmation on-chain
  (séquence ≥ 0, timeout 30 s). Un nom déjà enregistré est refusé
  côté client ET côté relay.
- **update** : le fichier de records est l'unique source de vérité.
  La séquence (on-chain + 1) et le `record_hash` (BLAKE3 canonique
  des records du fichier) sont **dérivés**, jamais saisis : signataire
  et vérificateurs ne peuvent pas diverger. Après confirmation de
  l'UpdateDomain, le `SignedDnsRecord` (records du fichier + owner +
  signature Ed25519 sur l'encodage canonique) est publié dans la
  DHT via `put_record` (qui revérifie contre la chaîne). Un domaine
  non enregistré, une identité non-propriétaire ou une erreur de
  fichier échouent proprement, sans toucher à la chaîne.

#### Fichier de records

Texte, un record par ligne, `#` = commentaire, lignes vides
ignorées ; l'encodage canonique trié rend l'ordre du fichier sans
effet sur le hash engagé. Le fichier est le set COMPLET (un update
remplace, il ne fusionne pas). Types :

```text
A 192.0.2.1
AAAA 2001:db8::1
CNAME www.example.uip
NS ns1.example.uip
MX 10 mail.example.uip
TXT texte libre (espaces normalisés)
```

Erreurs typées : type inconnu, IP/nom invalide (avec numéro de
ligne), set vide, doublon, > `MAX_RECORDS_PER_SET`.

## `--genesis` (fondateurs)

`scone relay --genesis genesis.bin` — **nœuds fondateurs uniquement**
(campagne kickstarter ; le fichier est produit par le binaire
`scone-genesis`). Règles normatives :

1. Le fichier est refusé **avant lecture** s'il dépasse 1 Mio, puis
   décodé strictement (`decode_complete` : aucun octet traînant).
   Le hash du bloc 0 est **recalculé**, jamais lu du fichier.
2. Validation sémantique : hauteur 0, `consensus` commençant par le
   network id du réseau demandé (`--network`), aucune transaction,
   manifeste d'allocations canonique. Toute violation = erreur
   typée au démarrage (jamais de panic, le daemon ne démarre pas).
3. Data-dir **vierge** : la chaîne naît sur la genèse allouée — les
   allocations sont appliquées en RAM **avant toute écriture disque**,
   puis persistées (états seedés + bloc genèse en clé méta). Le
   premier bloc produit ne peut donc qu'étendre la genèse allouée.
4. Data-dir **déjà productif** : la genèse de naissance du store est
   le `prev_hash` du bloc 1 (auto-certifiant). Hash du fichier égal =
   OK silencieux (l'option est **redondante après le premier
   démarrage**) ; différent = refus typé — un nœud rejoint un réseau
   par sync, jamais par `--genesis`.
5. Redémarrage **sans** l'option : un data-dir né fondateur retrouve
   sa genèse allouée via la clé méta (rebind RAM à chaque boot).

L'épingle de hash de genèse constante (anti-divergence réseau) est
un chantier séparé.

## Limites et garanties mémoire

| Ressource | Borne |
|---|---|
| Trame P2P | `MAX_MESSAGE_LEN` (1 Mio) avant parsing |
| Mempool | 4096 tx (rejet typé au-delà) ; `MAX_PENDING_PER_DOMAIN = 4` tx pendantes par domaine |
| Grace mempool | `MAX_GRACE_ATTEMPTS = 10` réinsertions d'une tx flushée pour raison non définitive (≈ 7 min à 40 s/cycle), puis éviction |
| Parking fork | `PARKED_CAP = 2048` blocs compétiteurs |
| Checkpoints en accumulation | `MAX_PENDING_CHECKPOINTS = 256` (éviction : plus petit hash de data) |
| Waiters DHT simultanés | `MAX_DHT_WAITERS` (256) ; lookup DHT borné 30 s (`DHT_LOOKUP_TIMEOUT`) |
| Blocs par réponse sync | 1 par aller-retour ; ≤ 10 000 lots par pair (`MAX_SYNC_ROUNDS`) ; borne protocol `MAX_BLOCKS_PER_REQUEST = 128` à la demande |
| Record DHT | `MAX_DHT_CACHE_ENTRY` (64 Kio) |
| Cache DHT persistant (disque) | double borne LRU : `DHT_CACHE_MAX_ENTRIES` (100 000 entrées) ET `DHT_CACHE_MAX_BYTES` (2 Gio) — la première atteinte évince par ordre de clé ; éviction GC dans la MÊME transaction que le bloc (un domaine tué par le GC chaîne ne laisse jamais d'entrée orpheline) |
| Cache négatif DHT | `DHT_NEGATIVE_TTL_SECS` (30 s), cap FIFO `DHT_NEGATIVE_CAP` (4 096) |
| Chaîne en RAM | fenêtre genesis + tip (historique servi par le store) |

Règle anti-boucle (H2) : une transaction n'est re-diffusée que si elle
vient d'être **réellement insérée** au mempool (première vue) ; un
doublon (déjà en pool ou déjà en chaîne) n'est jamais re-relayé. Un
bloc déjà canonique est détecté et ignoré avant `push_block` (aucun
re-store, aucun re-broadcast).

Décisions ouvertes : identité réseau persistante du relay, gossip
de diffusion (actuellement : request-response unicast vers chaque
pair connu). Résolus depuis la rédaction initiale : fork choice
réel (parking + reconsider_parked, voir ci-dessus), consensus PoW
(producteur signé + comité anchor), DNS resolver (surface `--dns`,
voir dns.md), canal de contrôle (unification RPC→p2p : plus de
TCP JSON).

## Arrêt propre (SIGTERM / SIGINT)

Le daemon `scone relay` (foreground) intercepte SIGTERM et SIGINT via
`tokio::signal` : au signal, la future `Relay::run` est annulée
(`tokio::select!` dans le binaire), ce qui drop le Swarm libp2p et ferme
les sockets QUIC. Le runtime tokio est ensuite arrêté avec un shutdown
borné (≤ 2 s) qui drop les tâches DNS détachées (UDP/TCP) et referme
leurs sockets — l'objectif est qu'un `kill` ne laisse aucune socket UDP
orpheline (cas observé sur la testnet 60). Un arrêt par signal logge
`relay shutting down (signal)` (INFO, stderr) et termine avec l'exit
code **0** : un arrêt demandé n'est pas un échec. Si l'installation
d'un handler échouait, le comportement par défaut du signal s'applique
(dégradation, jamais de panic).

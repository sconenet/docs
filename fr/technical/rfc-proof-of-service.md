# RFC — Preuve d'exploitation et comité de finalité (proof-of-service)

> Statut : IMPLÉMENTÉE (étapes 1-3 : core/protocol/blockchain/network ;
> l'émission automatique de scone-network est en place). Étape 4
> partielle : `tx sign`/`tx verify` acceptent un heartbeat, mais ni
> `tx build heartbeat` ni l'affichage du nombre d'opérateurs frais
> dans `scone status` n'existent. Étape 5 (RFC → forme normative
> consolidée) reste à faire.
> Date : octobre 2026. Perturbe : `scone-core`, `scone-protocol`,
> `scone-blockchain`, `scone-network`.

## 1. Motivation

Le pool de finalité actuel est le pool d'owners (propriétaires de
domaines/TLD vivants). Posséder donne un siège — **opérer** ne donne
rien. Cette RFC lie le droit de signer les checkpoints à une preuve
d'exploitation d'un relais en ligne, sans jamais conditionner la
possession :

- **Liberté préservée** : un TLD privé reste immortel et assignable
  sans condition d'activité. Seule la *signature de checkpoint* se
  gagne par la présence.
- **Incitation à opérer** : exploiter un TLD privé (ou tout siège)
  = siéger au comité. L'exploitant participe au maintien du réseau
  et le protocole le reconnaît.
- **Décentralisation mesurable** : le comité devient l'ensemble des
  opérateurs *effectivement en ligne*, pas des clés dormantes.

## 2. Vue d'ensemble

Une nouvelle transaction légère `Heartbeat`, signée par une clé
d'**opérateur**, émise une fois par époque. L'état tient une fenêtre
glissante des heartbeats récents. Le comité d'une époque est élu
**dans le sous-ensemble du pool dont l'opérateur a un heartbeat
frais**. Sans heartbeat : pas de signature, mais aucune perte de
droits de propriété.

## 3. La transaction Heartbeat

```
Heartbeat {
    operator: PublicKey,     // clé Ed25519 du relais opérateur
    tip_seen: BlockHash,     // dernier tip vu par le nœud
    height_seen: u64,        // hauteur correspondante
    timestamp: u64,          // Unix secondes
} + signature Ed25519 sur le payload canonique (réseau injecté)
```

Propriétés :
- ~120 octets encodés ; **aucun PoW** — le coût est la présence
  continue (un heartbeat par époque, pendant des mois).
- `tip_seen` sert à l'observabilité (détection d'opérateurs bogués
  ou attardés) ; il n'influence PAS la validité (un nœud en retard
  léger reste un nœud vivant).

### 3.1 Validation (règles d'acceptation)

1. Signature Ed25519 valide sur le payload canonique du réseau, et
   `owner` == identité dérivée de `operator` (`InvalidOwner` sinon).
2. `timestamp` strictement croissant par opérateur à travers les
   époques (anti-rejeu d'un ancien heartbeat signé — rejet typé
   `StaleHeartbeat` à l'application).
3. **Un seul heartbeat accepté par (`operator`, époque)** — un
   doublon dans la même époque est rejeté typé
   (`HeartbeatDuplicate`). L'époque est celle du bloc porteur
   (`hauteur / epoch_min_blocks`, même définition que la finalité).

### 3.2 Anti-spam

- 1 tx par opérateur par époque — le flux maximum est
  `taille_du_pool / epoch_secs`, négligeable.
- Pas de frais (pré-économie) : le taux est borné structurellement.

## 4. Délégation d'exploitation

Le lien siège → opérateur :

- `SetTldOperator { tld_id, operator }` (signée par le propriétaire
  du TLD privé ; un TLD ouvert n'a personne pour signer
  légitimement) — `operator: None` = révocation, retour à
  l'auto-opération par la clé propriétaire ; révocable
  instantanément.
- Pour les domaines : `SetDomainOperator { domain_id, operator }`
  (signée par le propriétaire du domaine). Même sémantique.
- `RegisterTld` / `RegisterDomain` / `AssignDomain` ne portent PAS
  de champ `operator` : la délégation se fait par les deux
  transactions dédiées ci-dessus (le lien siège → opérateur est
  toujours un fait on-chain récent, résolu au tip).

L'état maintient :

```
seat_operators: HashMap<OwnerId, OperatorKey>   // par siège
operator_seats: HashMap<OperatorKey, usize>     // compteur borné
```

**Borne anti-location** : un opérateur ne peut être délégué que par
**un seul siège** à la fois (`MAX_SEATS_PER_OPERATOR = 1`). Un
deuxième siège qui délègue vers un opérateur déjà pris → rejet typé
`OperatorAlreadyDelegated`. Rationale : sans borne, un gros
opérateur vend de la signature en masse et recrée de la
centralisation — exactement ce que le comité élu veut éviter.

## 5. Fenêtre de fraîcheur et élection

### 5.1 Fenêtre

```
recent_heartbeats: BTreeMap<(epoch, OperatorKey), ()>
```

- Entrée insérée à l'application d'un heartbeat valide.
- Purge déterministe à chaque application de bloc : les entrées
  sorties de la fenêtre `HEARTBEAT_WINDOW_EPOCHS = 2` époques sont
  retirées (journaled — annulable en reorg, comme les files
  d'expiration). Avec 2 : l'époque courante + la précédente — un
  opérateur peut rater UNE époque sans perdre sa fraîcheur, la
  deuxième l'expire.
- « Frais » = présence d'une entrée pour l'opérateur dans la
  fenêtre courante.

### 5.2 Élection du comité (modification de `select_committee`)

```
eligible(pool) = { sièges du pool dont l'opérateur associé
                   (ou la clé du owner si auto-opéré)
                   a un heartbeat frais }

si |eligible| >= MIN_FINALITY_COMMITTEE_SIZE :
    committee = select_committee(seed, eligible)   // tirage existant
sinon :
    committee = select_committee(seed, pool)       // dégradation sûre
```

Détails d'implémentation (vérifiés dans `finality.rs`, spécifiés
dans `/docs/fr/technical/blockchain.md` § « Sièges et comité de
finalité ») : le `pool` est le pool **gelé au dernier checkpoint
finalisé** (`FinalizedBase.pool` — jamais l'état vivant), la
`seed` dérive du hash de contenu du checkpoint précédent
(`signing_hash`), chaque siège est résolu en opérateur EFFECTIF
par l'état au tip (délégation = fait on-chain récent) et participe
sous sa clé effective, les clés bannies restent exclues.

La **dégradation** est explicite et sûre : un réseau au démarrage
(aucun heartbeat), ou une panne générale des opérateurs, retombe sur
le pool entier — la finalité n'est jamais perdue par construction.
C'est la règle « jamais de blocage » du projet appliquée ici.

### 5.3 Bootstrap froid

Époque 0 (et tant que moins de `MIN_FINALITY_COMMITTEE_SIZE`
opérateurs sont frais) : pool entier. La contrainte se resserre
naturellement à mesure que les opérateurs émettent. Aucune
période transitoire spéciale à configurer.

## 6. Émission automatique (scone-network)

Le relais émet un heartbeat par époque quand :

1. une clé anchor est armée (`--anchor-key` ou auto), ET
2. son identité (ou son délégué) détient un siège vivant.

L'émission se fait au premier cycle de production de l'époque —
aucun timer nouveau, ça roule avec `Wake::Produce`. Le heartbeat
n'est PAS produit dans un bloc par l'émetteur lui-même
nécessairement : c'est une tx comme les autres (mempool, production
par quiconque).

## 7. Observabilité

- Prévu, pas encore exposé : `operator_status` — par siège,
  l'opérateur délégué, la fraîcheur (frais / expiré / jamais vu),
  dernier `tip_seen` (le RPC de contrôle local a été retiré ;
  surface à définir côté p2p).
- `scone status` n'affiche pas encore le nombre d'opérateurs
  frais / la taille du pool (les données existent dans l'état :
  `fresh_operators`).
- Argument campagne : « exploiter un TLD privé = siéger au comité
  de finalité du DNS décentralisé » — vérifiable sur chaîne.

## 8. Ce que cette RFC ne fait PAS

- **Aucune perte de possession** pour inactivité (l'option A —
  dégénérescence du TLD non exploité — est explicitement rejetée).
- **Aucune récompense** (l'option C attend une couche économique).
- Le heartbeat ne prouve pas la *qualité* du service (blocs
  produits, records DHT servis) — seulement la présence. Compter la
  contribution réelle est une extension future possible, plus
  jouable.

## 9. Invariants protégés

- Déterminisme bit-exact : la fenêtre et l'élection sont des
  fonctions pures de l'état appliqué (journaled, undo bit-exact).
- Jamais de panic sur données réseau : heartbeat malformé → rejet
  typé (`InvalidOwner`, `StaleHeartbeat`, `HeartbeatDuplicate`,
  `OperatorAlreadyDelegated`).
- La possession est inconditionnelle ; seule la signature de
  checkpoint est conditionnelle.

## 10. Plan d'implémentation

| Étape | Crate | Contenu | Taille | État |
|---|---|---|---|---|
| 1 | core + protocol | `Heartbeat`, `SetTldOperator`, `SetDomainOperator`, wire, validation | M | fait |
| 2 | blockchain | fenêtre glissante (journaled), pool filtré, élection, ~15 tests | L | fait |
| 3 | network | émission automatique, relay de tx | S | fait |
| 4 | CLI | `tx build heartbeat` (debug), affichage status | S | partiel (`tx sign`/`verify` seulement) |
| 5 | docs | cette RFC → normative, mise à jour protocol.md/finality | S | à faire |

## 11. Décisions ouvertes (à trancher par Benoît)

1. `HEARTBEAT_WINDOW_EPOCHS = 2` — confortable ou trop lâche ?
   (2 époques = un opérateur peut rater une époque sans perdre son
   siège de signature.) *C'est la valeur codée aujourd'hui.*
2. `MAX_SEATS_PER_OPERATOR = 1` — ou tolérer 2-3 pour les petits
   opérateurs qui hébergent plusieurs namespaces ? *Valeur codée : 1.*
3. Exposer `operator_status` en RPC dès l'étape 3 ou plus tard ?
   *Devenu mou : le RPC de contrôle local a été retiré (nœud
   p2p-only) — la surface reste à définir.*

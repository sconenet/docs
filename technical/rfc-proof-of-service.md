# RFC — Preuve d'exploitation et comité de finalité (proof-of-service)

> Statut : PROPOSITION — à valider avant implémentation.
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

1. Signature Ed25519 valide sur le payload canonique du réseau.
2. `timestamp` dans une fenêtre raisonnable du bloc porteur
   (± `epoch_secs` — borne anti-replay historique).
3. **Un seul heartbeat accepté par (`operator`, époque)** — un
   doublon dans la même époque est rejeté typé
   (`HeartbeatDuplicate`). Le mempool applique la même exclusion
   en attente (remplace l'ancien non inclus).
4. `timestamp` strictement croissant par opérateur à travers les
   époques (anti-rejeu d'un ancien heartbeat signé).

### 3.2 Anti-spam

- 1 tx par opérateur par époque — le flux maximum est
  `taille_du_pool / epoch_secs`, négligeable.
- Pas de frais (pré-économie) : le taux est borné structurellement.

## 4. Délégation d'exploitation

Le lien siège → opérateur :

- `RegisterTld` / `RegisterDomain` / `AssignDomain` gagnent un champ
  optionnel `operator: Option<PublicKey>`. Défaut : `None` =
  auto-opération par la clé propriétaire.
- Nouvelle transaction `SetTldOperator { tld, operator }` (signée
  par le propriétaire du TLD) — révocable instantanément.
- Pour les domaines : `SetDomainOperator { domain, operator }`
  (signée par le propriétaire du domaine). Même sémantique.

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
- Purge déterministe à chaque frontière d'époque : les entrées plus
  vieilles que `HEARTBEAT_WINDOW_EPOCHS = 2` époques sont retirées
  (journaled — annulable en reorg, comme les files d'expiration).
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

- RPC `operator_status` : par siège, l'opérateur délégué, la fraîcheur
  (frais / expiré / jamais vu), dernier `tip_seen`.
- `scone status` affiche le nombre d'opérateurs frais / taille du
  pool.
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
  typé (`InvalidHeartbeat`, `HeartbeatDuplicate`,
  `OperatorAlreadyDelegated`).
- La possession est inconditionnelle ; seule la signature de
  checkpoint est conditionnelle.

## 10. Plan d'implémentation

| Étape | Crate | Contenu | Taille |
|---|---|---|---|
| 1 | core + protocol | `Heartbeat`, `SetTldOperator`, `SetDomainOperator`, wire, validation | M |
| 2 | blockchain | fenêtre glissante (journaled), pool filtré, élection, ~15 tests | L |
| 3 | network | émission automatique, relay de tx | S |
| 4 | CLI | `tx build heartbeat` (debug), affichage status | S |
| 5 | docs | cette RFC → normative, mise à jour protocol.md/finality | S |

## 11. Décisions ouvertes (à trancher par Benoît)

1. `HEARTBEAT_WINDOW_EPOCHS = 2` — confortable ou trop lâche ?
   (2 époques = un opérateur peut rater une époque sans perdre son
   siège de signature.)
2. `MAX_SEATS_PER_OPERATOR = 1` — ou tolérer 2-3 pour les petits
   opérateurs qui hébergent plusieurs namespaces ?
3. Exposer `operator_status` en RPC dès l'étape 3 ou plus tard ?

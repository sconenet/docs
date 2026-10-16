# RFC — Preuve d'exploitation et comité de finalité (proof-of-service)

> Statut : NORMATIF — implémenté de bout en bout (étapes 1 à 5 :
> core/protocol/blockchain/network/CLI ; ce document est la
> forme consolidée). Chaque règle ci-dessous est vérifiée contre le code ;
> les constantes citées sont les valeurs codées.
> Date : octobre 2026 (consolidation normative). Touche :
> `scone-core`, `scone-protocol`, `scone-blockchain`,
> `scone-network`, `scone` (CLI).

## 1. Motivation

Le pool de finalité est le pool des propriétaires (propriétaires de
domaines/TLDs vivants). Posséder donne un siège — **exploiter** ne donne
rien. Cette RFC lie le droit de signer les points de contrôle à une
preuve d'exploitation d'un relais en ligne, sans jamais conditionner la
propriété :

- **Liberté préservée** : un TLD privé reste attribuable sans condition
  d'activité aussi longtemps que son terme est renouvelé
  (`RenewTld`, terme de 3 ans). Seule la *signature des points de
  contrôle* se gagne par la présence.
- **Incitation à exploiter** : exploiter un TLD privé (ou n'importe quel
  siège) = siéger au comité. L'opérateur participe au maintien en ligne
  du réseau et le protocole le reconnaît.
- **Décentralisation mesurable** : le comité devient l'ensemble des
  opérateurs *réellement en ligne*, pas des clés dormantes.

## 2. Vue d'ensemble

Une transaction `Heartbeat` légère (discriminant `0x71`),
signée par une clé d'**opérateur**, émise une fois par époque. L'état
conserve une fenêtre glissante des battements de cœur récents. Le comité
d'une époque est élu **sur le sous-ensemble du pool dont l'opérateur
effectif a un battement de cœur frais**. Sans battement de cœur : pas de
signature, mais aucune perte de droits de propriété.

## 3. La transaction Heartbeat

```
Heartbeat {
    operator: PublicKey,     // clé Ed25519 de l'opérateur (le signataire)
    owner: OwnerId,          // identité RE-DÉRIVÉE de l'opérateur (jamais confiance)
    tip_seen: [u8; 32],      // dernier sommet vu par le nœud
    height_seen: u64,        // hauteur correspondante
    timestamp: u64,          // secondes Unix
} + signature Ed25519 sur la charge canonique (réseau injecté)
```

Corps signé (wire, `scone-protocol`) :
`operator[32] ‖ owner[32] ‖ tip_seen[32] ‖ height_seen.v ‖
timestamp.v ‖ operator[32]` (la clé du signataire apparaît en queue
du corps, comme pour chaque tx) `‖ signature[64]`, soit ~230 octets
encodés — **pas de PoW** : le coût est la présence continue (un
battement de cœur par époque, pendant des mois).

Le signataire EST l'opérateur : `owner` est toujours re-dérivé de
`operator` (jamais pris de l'entrée), et le décodage rejette avec une
erreur typée (`InvalidOwner`) une clé de signature qui diffère du champ
`operator` de tête.

- `tip_seen`/`height_seen` servent l'observabilité (détection
  d'opérateurs bogués ou en retard) ; ils n'influencent PAS la validité
  (un nœud légèrement en retard reste un nœud vivant).

### 3.1 Validation pure (scone-core)

1. Signature Ed25519 valide sur la charge canonique du réseau.
2. `owner` == identité dérivée de `operator` (`InvalidOwner`
   sinon) — vérifié à l'encodage, au décodage ET par `validate()`.

### 3.2 Règles d'état (application, scone-blockchain)

À l'application d'un battement de cœur dans un bloc porteur de hauteur
`block_height` (chaque rejet précède toute mutation) :

1. **Anti-replay inter-époques** : `timestamp` strictement croissant
   par opérateur — rejet typé `StaleHeartbeat` si
   `timestamp <= operator_last_hb[operator]`.
2. **Unicité par époque** : un seul battement de cœur accepté par
   (`operator`, époque du bloc porteur) — le doublon est rejeté
   avec une erreur typée (`HeartbeatDuplicate`). L'époque est
   `block_height / epoch_min_blocks` (même définition que la finalité,
   jamais une nouvelle constante).
3. Insertion journalisée dans la fenêtre + mise à jour de
   `operator_last_hb` (réversible en réorg, undo bit-exact).

### 3.3 Anti-spam

- 1 tx acceptée par opérateur par époque — le flux maximum est
  `pool_size / epoch_secs`, négligeable.
- Pas de frais (pré-économie) : le débit est structurellement borné.

## 4. Délégation d'exploitation

Le lien siège → opérateur passe par deux transactions dédiées :

- `SetTldOperator { tld_id, operator }` (discriminant `0xA9`),
  signée par le propriétaire **actuel** du TLD **privé** (un TLD
  ouvert n'a personne pour signer légitimement — rejet typé
  `TldIsOpen`). `operator: None` = révocation, retour à
  l'auto-exploitation par la clé du propriétaire ; révocable
  instantanément.
- `SetDomainOperator { domain_id, operator }` (discriminant
  `0x5E`), signée par le propriétaire actuel du domaine. Mêmes
  sémantiques.
- `RegisterTld` / `RegisterDomain` / `AssignDomain` ne portent AUCUN
  champ `operator` : le lien siège → opérateur est toujours un fait
  on-chain récent, résolu au sommet.

L'option opérateur sur le wire est canonique : `0x00` = `None`
(auto-exploitation), `0x01` = clé brute de 32 octets ; toute autre
valeur est rejetée au décodage.

L'état maintient :

```
seat_operators: HashMap<OwnerId, OperatorKey>   // par siège
operator_seats: HashMap<OperatorKey, u64>       // compteur borné
operator_last_hb: HashMap<OperatorKey, u64>     // dernier ts accepté
```

Règles d'application (`apply_seat_operator`, journalisées) :

- **Idempotence** : re-déléguer au même opérateur (ou révoquer une
  délégation inexistante) est un no-op non journalisé (l'undo
  bit-exact l'exige).
- **Borne anti-location** : un opérateur ne peut être délégué que par
  **un seul siège** à la fois (`MAX_SEATS_PER_OPERATOR = 1`). Un
  second siège déléguant vers un opérateur déjà pris → rejet typé
  `OperatorAlreadyDelegated`. Justification : sans borne, un gros
  opérateur vend la signature en gros et recrée la centralisation —
  exactement ce que le comité élu est censé éviter.

## 5. Fenêtre de fraîcheur

### 5.1 Fenêtre glissante

```
recent_heartbeats: BTreeMap<(epoch, OperatorKey), ()>
```

- Entrée insérée à l'application d'un battement de cœur valide (§3.2).
- Purge déterministe à CHAQUE application de bloc
  (`gc_heartbeats_journaled`) : toute entrée d'époque `e` avec
  `e + HEARTBEAT_WINDOW_EPOCHS <= epoch` est retirée (journalisée —
  réversible en réorg, comme les files d'expiration).
- `HEARTBEAT_WINDOW_EPOCHS = 2` : l'époque courante + la précédente —
  un opérateur peut manquer UNE époque sans perdre la fraîcheur,
  la seconde l'expire.
- **Cap mémoire structurel** : la fenêtre contient au plus
  `HEARTBEAT_WINDOW_EPOCHS × operator_count` entrées (une par
  opérateur par époque, clé en tuple) ; `operator_last_hb` est borné
  par le nombre d'opérateurs et n'est jamais purgé (la règle
  d'anti-replay est monotone).

### 5.2 Fraîcheur

« Frais » = présence d'une entrée `(e, operator)` pour `e` dans
`[epoch − HEARTBEAT_WINDOW_EPOCHS + 1, epoch]`. Comme l'état purge à
chaque bloc, la présence dans `recent_heartbeats` EST la fraîcheur
(`operator_is_fresh`).

### 5.3 Opérateur effectif

L'**opérateur effectif** d'un siège est `seat_operators[seat]` si une
délégation vivante existe, sinon la clé du propriétaire
(`effective_operator`). Le siège signe sous cette clé effective — une
délégation déplace le droit de signature, jamais la propriété.

## 6. Élection du comité

```
eligible = { opérateurs effectifs des sièges du pool figé
             (clé bannie exclue) ayant un battement de cœur frais }

if |eligible| >= MIN_FINALITY_COMMITTEE_SIZE (= 4) :
    committee = select_committee(seed, eligible)   // tirage existant
else :
    committee = select_committee(seed, pool)       // dégradation sûre
```

Détails normatifs (vérifiés dans `finality.rs`, spécifiés dans
`/docs/en/technical/blockchain.md` § « Sièges et comité de
finalité ») :

- le `pool` est le pool **figé au dernier point de contrôle
  finalisé** (`FinalizedBase.pool` — jamais l'état vivant) : une vue
  locale divergerait entre nœuds et le quorum ne se formerait
  jamais ;
- la `seed` dérive du hachage de contenu du point de contrôle
  précédent (`signing_hash`) ;
- chaque siège du pool figé est résolu en son opérateur EFFECTIF par
  l'état au sommet (délégation et battements de cœur sont des faits
  on-chain récents : deux nœuds au même sommet calculent le même
  ensemble) ; les clés bannies restent exclues ;
- le tirage court sur l'ensemble trié/dédupliqué des clés effectives
  (`select_committee` inchangé).

**Dégradation explicite et sûre** : un réseau en amorçage (pas de
battement de cœur), ou une panne générale des opérateurs, retombe sur
le pool entier — la finalité n'est jamais perdue par construction.
C'est la règle « jamais d'interblocage » du projet appliquée ici.

### 6.1 Amorçage à froid

Époque 0 (et aussi longtemps que moins de `MIN_FINALITY_COMMITTEE_SIZE`
opérateurs sont frais) : le pool entier. La contrainte se resserre
naturellement à mesure que les opérateurs émettent. Aucune période
transitoire particulière à configurer.

## 7. Émission automatique (scone-network)

À chaque cycle de production (`Wake::Produce` — pas de nouveau
minuteur), le relais vérifie si une LIMITE D'ÉPOQUE a été franchie
(`height / epoch_min_blocks`, même définition que la finalité). Au
premier cycle d'une nouvelle époque, il émet un battement de cœur si :

1. une clé d'ancre est armée (`--anchor-key` ou auto), ET
2. cette clé détient elle-même un siège vivant (contrôle LOCAL : la clé
   est dans le pool d'éligibilité on-chain — propriétaires de
   domaines/TLDs vivants, clés bannies exclues).

Anti-spam local : au plus UNE émission par époque par nœud
(`last_emitted_epoch` mémo, mémoire locale, jamais état de chaîne) ; le
mémo n'est pris qu'après une injection réussie dans le mempool — un
échec réessaie au cycle suivant. Rien ici n'est fatal : au pire le
battement de cœur attend le cycle suivant.

Un **opérateur délégué** (qui n'est pas lui-même propriétaire) n'est pas
couvert par l'émission automatique : il émet via le constructeur CLI du
§8 (exploitation manuelle ou cloisonnée). Le battement de cœur n'est
PAS nécessairement produit dans un bloc par l'émetteur lui-même : c'est
une tx comme une autre (mempool, production par n'importe qui).

## 8. CLI (`scone`)

Le constructeur de battement de cœur hors ligne (retiré avec la surface
`tx`) a été supprimé avec la surface tx hors ligne (cli-simplify-v1).
Les relais émettent leur propre battement de cœur automatiquement (§7) ;
une émission manuelle/cloisonnée, si jamais besoin à nouveau,
reviendrait en commande dédiée. Les défauts étaient : `tip_seen` = 32
octets zéro (observabilité uniquement), `height_seen` = 0,
`timestamp` = now.
- Comme pour chaque tx, `tx sign` **relie** opérateur et propriétaire à
  la VRAIE clé de signature (les valeurs d'entrée ne sont jamais crues) ;
  `tx verify` valide localement.

## 9. Observabilité

- `NodeStatus` (message wire p2p `0x0d`) porte
  `fresh_operators: u32` (varint) : le nombre d'opérateurs ayant un
  battement de cœur frais — le sous-ensemble vivant du pool éligible au
  comité (opérateurs effectifs des sièges du pool, clés bannies
  exclues, triés/dédupliqués). Rempli par le relais depuis
  `state().fresh_operators(height)` ; affiché par `scone status`
  sous la forme `fresh operators: N`.
- Argument campagne : « exploiter un TLD privé = siéger au comité de
  finalité du DNS décentralisé » — vérifiable
  on-chain.
- Surface future (non spécifiée) : `operator_status` — par siège,
  l'opérateur délégué, la fraîcheur (frais / expiré / jamais vu),
  le dernier `tip_seen`. Le RPC de contrôle local a été supprimé
  (nœud p2p uniquement) ; la surface reste à définir.

## 10. Ce que cette RFC ne fait PAS

- **Aucune perte de possession** pour inactivité (l'option A — déclin du
  TLD non exploité — est explicitement rejetée).
- **Aucune récompense** (l'option C attend une couche économique).
- Le battement de cœur ne prouve pas la *qualité* du service (blocs
  produits, enregistrements DHT servis) — seulement la présence.
  Compter la contribution réelle est une extension future possible,
  plus jouable.

## 11. Invariants protégés

- Déterminisme bit-exact : la fenêtre et l'élection sont des fonctions
  pures de l'état appliqué (journalisées, undo bit-exact).
- Jamais de panic sur des données réseau : battement de cœur ou
  délégation malformés → rejets typés (`InvalidOwner`,
  `StaleHeartbeat`, `HeartbeatDuplicate`,
  `OperatorAlreadyDelegated`, `TldIsOpen`, `NotTldOwner`,
  `NotOwner`).
- La possession est inconditionnelle ; seule la signature des points de
  contrôle est conditionnelle.

## 12. Constantes et paramètres (valeurs codées)

| Nom | Valeur | Emplacement |
|---|---|---|
| `HEARTBEAT_WINDOW_EPOCHS` | 2 | `scone-blockchain/src/state.rs` |
| `MAX_SEATS_PER_OPERATOR` | 1 | `scone-blockchain/src/state.rs` |
| `MIN_FINALITY_COMMITTEE_SIZE` | 4 | `scone-core/src/consensus_params.rs` |
| Discriminant `Heartbeat` | `0x71` | `scone-protocol/src/transaction.rs` |
| Discriminant `SetTldOperator` | `0xA9` | idem |
| Discriminant `SetDomainOperator` | `0x5E` | idem |

Ces valeurs font partie du consensus : les changer casse la rejouabilité
de la chaîne (fenêtre/purge) ou l'élection (comité).

## 13. Plan d'implémentation — état final

| Étape | Crate | Contenu | État |
|---|---|---|---|
| 1 | core + protocol | `Heartbeat`, `SetTldOperator`, `SetDomainOperator`, wire, validation | fait |
| 2 | blockchain | fenêtre glissante (journalisée), pool filtré, élection, ~15 tests | fait |
| 3 | network | émission automatique, relais de tx | fait |
| 4 | CLI | `tx build heartbeat`, affichage `fresh operators` dans `scone status` | fait |
| 5 | docs | cette RFC consolidée en forme normative, miroir fr/en | fait |

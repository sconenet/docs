# RFC — Preuve d'exploitation et comité de finalité (proof-of-service)

> Statut : NORMATIVE — implémentée de bout en bout (étapes 1 à 5 :
> core/protocol/blockchain/network/CLI ; cette document est la forme
> consolidée). Chaque règle ci-dessous est vérifiée contre le code ;
> les constantes citées sont les valeurs codées.
> Date : octobre 2026 (consolidation normative). Perturbe :
> `scone-core`, `scone-protocol`, `scone-blockchain`, `scone-network`,
> `scone` (CLI).

## 1. Motivation

Le pool de finalité est le pool d'owners (propriétaires de
domaines/TLD vivants). Posséder donne un siège — **opérer** ne donne
rien. Cette RFC lie le droit de signer les checkpoints à une preuve
d'exploitation d'un relais en ligne, sans jamais conditionner la
possession :

- **Liberté préservée** : un TLD privé reste assignable sans
  condition d'activité aussi longtemps que son terme est renouvelé
  (`RenewTld`, terme de 3 ans). Seule la *signature de checkpoint*
  se gagne par la présence.
- **Incitation à opérer** : exploiter un TLD privé (ou tout siège)
  = siéger au comité. L'exploitant participe au maintien du réseau
  et le protocole le reconnaît.
- **Décentralisation mesurable** : le comité devient l'ensemble des
  opérateurs *effectivement en ligne*, pas des clés dormantes.

## 2. Vue d'ensemble

Une transaction légère `Heartbeat` (discriminant `0x71`), signée par
une clé d'**opérateur**, émise une fois par époque. L'état tient une
fenêtre glissante des heartbeats récents. Le comité d'une époque est
élu **dans le sous-ensemble du pool dont l'opérateur effectif a un
heartbeat frais**. Sans heartbeat : pas de signature, mais aucune
perte de droits de propriété.

## 3. La transaction Heartbeat

```
Heartbeat {
    operator: PublicKey,     // clé Ed25519 de l'opérateur (le signataire)
    owner: OwnerId,          // identité RE-DÉRIVÉE de operator (jamais crue)
    tip_seen: [u8; 32],      // dernier tip vu par le nœud
    height_seen: u64,        // hauteur correspondante
    timestamp: u64,          // Unix secondes
} + signature Ed25519 sur le payload canonique (réseau injecté)
```

Corps signé (wire, `scone-protocol`) :
`operator[32] ‖ owner[32] ‖ tip_seen[32] ‖ height_seen.v ‖
timestamp.v ‖ operator[32]` (la clé du signataire apparaît en fin de
corps, comme pour toutes les tx) `‖ signature[64]`, soit ~230 octets
encodés — **aucun PoW** : le coût est la présence continue (un
heartbeat par époque, pendant des mois).

Le signataire EST l'opérateur : `owner` est toujours re-dérivé de
`operator` (jamais pris de l'entrée), et le décodage rejette de façon
typée (`InvalidOwner`) une clé de signature différente du champ
`operator` de tête.

- `tip_seen`/`height_seen` servent à l'observabilité (détection
  d'opérateurs bogués ou attardés) ; ils n'influencent PAS la
  validité (un nœud en retard léger reste un nœud vivant).

### 3.1 Validation pure (scone-core)

1. Signature Ed25519 valide sur le payload canonique du réseau.
2. `owner` == identité dérivée de `operator` (`InvalidOwner` sinon) —
   vérifié à l'encodage, au décodage ET par `validate()`.

### 3.2 Règles d'état (application, scone-blockchain)

À l'application d'un heartbeat dans un bloc porteur de hauteur
`block_height` (tout rejet précède toute mutation) :

1. **Anti-rejeu inter-époques** : `timestamp` strictement croissant
   par opérateur — rejet typé `StaleHeartbeat` si
   `timestamp <= operator_last_hb[operator]`.
2. **Unicité par époque** : un seul heartbeat accepté par
   (`operator`, époque du bloc porteur) — doublon rejeté typé
   `HeartbeatDuplicate`. L'époque est `block_height /
   epoch_min_blocks` (même définition que la finalité, jamais une
   constante nouvelle).
3. Insertion journaled dans la fenêtre + mise à jour de
   `operator_last_hb` (annulables en reorg, undo bit-exact).

### 3.3 Anti-spam

- 1 tx acceptée par opérateur par époque — le flux maximum est
  `taille_du_pool / epoch_secs`, négligeable.
- Pas de frais (pré-économie) : le taux est borné structurellement.

## 4. Délégation d'exploitation

Le lien siège → opérateur se fait par deux transactions dédiées :

- `SetTldOperator { tld_id, operator }` (discriminant `0xA9`),
  signée par le propriétaire **actuel** du TLD **privé** (un TLD
  ouvert n'a personne pour signer légitimement — rejet typé
  `TldIsOpen`). `operator: None` = révocation, retour à
  l'auto-opération par la clé propriétaire ; révocable
  instantanément.
- `SetDomainOperator { domain_id, operator }` (discriminant `0x5E`),
  signée par le propriétaire actuel du domaine. Même sémantique.
- `RegisterTld` / `RegisterDomain` / `AssignDomain` ne portent PAS
  de champ `operator` : le lien siège → opérateur est toujours un
  fait on-chain récent, résolu au tip.

L'option opérateur sur le wire est canonique : `0x00` = `None`
(auto-opération), `0x01` = clé brute 32 octets ; toute autre valeur
est rejetée au décodage.

L'état maintient :

```
seat_operators: HashMap<OwnerId, OperatorKey>   // par siège
operator_seats: HashMap<OperatorKey, u64>       // compteur borné
operator_last_hb: HashMap<OperatorKey, u64>     // dernier ts accepté
```

Règles d'application (`apply_seat_operator`, journaled) :

- **Idempotence** : re-déléguer au même opérateur (ou révoquer une
  délégation inexistante) est un no-op non journalisé (l'undo
  bit-exact l'exige).
- **Borne anti-location** : un opérateur ne peut être délégué que par
  **un seul siège** à la fois (`MAX_SEATS_PER_OPERATOR = 1`). Un
  deuxième siège qui délègue vers un opérateur déjà pris → rejet
  typé `OperatorAlreadyDelegated`. Rationale : sans borne, un gros
  opérateur vend de la signature en masse et recrée de la
  centralisation — exactement ce que le comité élu veut éviter.

## 5. Fenêtre de fraîcheur

### 5.1 Fenêtre glissante

```
recent_heartbeats: BTreeMap<(epoch, OperatorKey), ()>
```

- Entrée insérée à l'application d'un heartbeat valide (§3.2).
- Purge déterministe à CHAQUE application de bloc
  (`gc_heartbeats_journaled`) : toute entrée d'époque `e` avec
  `e + HEARTBEAT_WINDOW_EPOCHS <= epoch` est retirée (journaled —
  annulable en reorg, comme les files d'expiration).
- `HEARTBEAT_WINDOW_EPOCHS = 2` : l'époque courante + la précédente
  — un opérateur peut rater UNE époque sans perdre sa fraîcheur, la
  deuxième l'expire.
- **Cap mémoire structurel** : la fenêtre contient au plus
  `HEARTBEAT_WINDOW_EPOCHS × nb_opérateurs` entrées (une par
  opérateur par époque, clé du tuple) ; `operator_last_hb` est
  bornée par le nombre d'opérateurs et n'est jamais purgée (la règle
  anti-rejeu est monotone).

### 5.2 Fraîcheur

« Frais » = présence d'une entrée `(e, operator)` pour `e` dans
`[epoch − HEARTBEAT_WINDOW_EPOCHS + 1, epoch]`. Comme l'état purge à
chaque bloc, la présence dans `recent_heartbeats` EST la fraîcheur
(`operator_is_fresh`).

### 5.3 Opérateur effectif

L'**opérateur effectif** d'un siège est `seat_operators[siège]` si
une délégation vivante existe, sinon la clé du owner
(`effective_operator`). C'est sous cette clé effectif que le siège
signe — une délégation déplace le droit de signature, jamais la
propriété.

## 6. Élection du comité

```
eligible = { opérateurs effectifs des sièges du pool gelé
             (clé bannie exclue) ayant un heartbeat frais }

si |eligible| >= MIN_FINALITY_COMMITTEE_SIZE (= 4) :
    committee = select_committee(seed, eligible)   // tirage existant
sinon :
    committee = select_committee(seed, pool)       // dégradation sûre
```

Détails normatifs (vérifiés dans `finality.rs`, spécifiés dans
`/docs/fr/technical/blockchain.md` § « Sièges et comité de
finalité ») :

- le `pool` est le pool **gelé au dernier checkpoint finalisé**
  (`FinalizedBase.pool` — jamais l'état vivant) : une vue locale
  divergerait entre nœuds et le quorum ne se formerait jamais ;
- la `seed` dérive du hash de contenu du checkpoint précédent
  (`signing_hash`) ;
- chaque siège du pool gelé est résolu en opérateur EFFECTIF par
  l'état au tip (délégation et heartbeats sont des faits on-chain
  récents : deux nœuds au même tip calculent le même ensemble) ;
  les clés bannies restent exclues ;
- le tirage se fait sur l'ensemble trié/dédupliqué des clés
  effectives (`select_committee` inchangé).

**Dégradation explicite et sûre** : un réseau au démarrage (aucun
heartbeat), ou une panne générale des opérateurs, retombe sur le
pool entier — la finalité n'est jamais perdue par construction.
C'est la règle « jamais de blocage » du projet appliquée ici.

### 6.1 Bootstrap froid

Époque 0 (et tant que moins de `MIN_FINALITY_COMMITTEE_SIZE`
opérateurs sont frais) : pool entier. La contrainte se resserre
naturellement à mesure que les opérateurs émettent. Aucune période
transitoire spéciale à configurer.

## 7. Émission automatique (scone-network)

À chaque cycle de production (`Wake::Produce` — aucun timer
nouveau), le relay vérifie le franchissement de frontière d'époque
(`height / epoch_min_blocks`, même définition que la finalité). Au
premier cycle d'une nouvelle époque, il émet un heartbeat si :

1. une clé anchor est armée (`--anchor-key` ou auto), ET
2. cette clé détient elle-même un siège vivant (contrôle LOCAL :
   la clé est dans le pool d'éligibilité on-chain — owners de
   domaines/TLD vivants, clés bannies exclues).

Anti-spam local : au plus UNE émission par époque par nœud (mémo
`last_emitted_epoch`, mémoire locale, jamais un état de chaîne) ;
le mémo n'est pris qu'après injection réussie au mempool — un échec
réessaie au cycle suivant. Rien ici n'est fatal : au pire le
heartbeat attend le cycle suivant.

Un **opérateur délégué** (qui n'est pas lui-même owner) n'est pas
couvert par l'émission automatique : il émet via le constructeur
CLI du §8 (opération manuelle ou air-gapped). Le heartbeat n'est
PAS nécessairement produit dans un bloc par l'émetteur lui-même :
c'est une tx comme les autres (mempool, production par quiconque).

## 8. CLI (`scone`)

```
scone tx build heartbeat --identity NOM [--tip_seen HEX32] [--height_seen N] [--timestamp TS]
```

- Constructeur offline de heartbeat (opération manuelle/air-gapped —
  le relay émet le sien automatiquement, §7). `--identity` nomme
  l'identité OPÉRATEUR destinée à signer ; défauts : `tip_seen` =
  32 octets nuls (le champ est observabilité-seule), `height_seen`
  = 0, `timestamp` = maintenant.
- Comme pour toutes les tx, `tx sign` **rebinde** operator et owner
  sur la VRAIE clé de signature (valeurs d'entrée jamais crues) ;
  `tx verify` valide localement.

## 9. Observabilité

- `NodeStatus` (message `0x0d` du wire p2p) porte
  `fresh_operators: u32` (varint) : nombre d'opérateurs avec un
  heartbeat frais — le sous-ensemble vivant du pool éligible au
  comité (opérateurs effectifs des sièges du pool, clés bannies
  exclues, trié/dédupliqué). Rempli par le relay depuis
  `state().fresh_operators(height)` ; affiché par `scone status`
  comme `fresh operators: N`.
- Argument campagne : « exploiter un TLD privé = siéger au comité
  de finalité du DNS décentralisé » — vérifiable sur chaîne.
- Surface future (non spécifiée) : `operator_status` — par siège,
  l'opérateur délégué, la fraîcheur (frais / expiré / jamais vu),
  dernier `tip_seen`. Le RPC de contrôle local a été retiré (nœud
  p2p-only) ; la surface reste à définir.

## 10. Ce que cette RFC ne fait PAS

- **Aucune perte de possession** pour inactivité (l'option A —
  dégénérescence du TLD non exploité — est explicitement rejetée).
- **Aucune récompense** (l'option C attend une couche économique).
- Le heartbeat ne prouve pas la *qualité* du service (blocs
  produits, records DHT servis) — seulement la présence. Compter la
  contribution réelle est une extension future possible, plus
  jouable.

## 11. Invariants protégés

- Déterminisme bit-exact : la fenêtre et l'élection sont des
  fonctions pures de l'état appliqué (journaled, undo bit-exact).
- Jamais de panic sur données réseau : heartbeat ou délégation
  malformés → rejets typés (`InvalidOwner`, `StaleHeartbeat`,
  `HeartbeatDuplicate`, `OperatorAlreadyDelegated`, `TldIsOpen`,
  `NotTldOwner`, `NotOwner`).
- La possession est inconditionnelle ; seule la signature de
  checkpoint est conditionnelle.

## 12. Constantes et paramètres (valeurs codées)

| Nom | Valeur | Lieu |
|---|---|---|
| `HEARTBEAT_WINDOW_EPOCHS` | 2 | `scone-blockchain/src/state.rs` |
| `MAX_SEATS_PER_OPERATOR` | 1 | `scone-blockchain/src/state.rs` |
| `MIN_FINALITY_COMMITTEE_SIZE` | 4 | `scone-core/src/consensus_params.rs` |
| Discriminant `Heartbeat` | `0x71` | `scone-protocol/src/transaction.rs` |
| Discriminant `SetTldOperator` | `0xA9` | idem |
| Discriminant `SetDomainOperator` | `0x5E` | idem |

Ces valeurs font partie du consensus : les changer casse la
rejouabilité de la chaîne (window/purge) ou l'élection (comité).

## 13. Plan d'implémentation — état final

| Étape | Crate | Contenu | État |
|---|---|---|---|
| 1 | core + protocol | `Heartbeat`, `SetTldOperator`, `SetDomainOperator`, wire, validation | faite |
| 2 | blockchain | fenêtre glissante (journaled), pool filtré, élection, ~15 tests | faite |
| 3 | network | émission automatique, relay de tx | faite |
| 4 | CLI | `tx build heartbeat`, affichage `fresh operators` dans `scone status` | faite |
| 5 | docs | cette RFC consolidée en forme normative, miroir fr/en | faite |

# Blockchain

Document de référence de la couche blockchain : identités, hachage,
genèse, état canonique, ordre des transactions et validation des blocs.
L'implémentation de référence est la crate
[`scone-blockchain`](../../../crates/scone-blockchain), au-dessus de
`scone-protocol` (encodage canonique), `scone-core` (règles métier) et
`scone-crypto` (BLAKE3).

Périmètre actuel : logique blockchain avec consensus réel (producteur
signé, points de contrôle, finalité), **état en mémoire** au-dessus d'un
backend KV (le stockage concret vit dans `scone-storage`) — voir
[Différé au consensus](#différé-au-consensus).

## Identités et séparation des domaines de hachage

Quatre identités, quatre rôles, quatre préfixes BLAKE3 distincts — un
préfixe n'est jamais réutilisé pour deux usages :

| Type | Rôle | Préfixe | Formule |
|---|---|---|---|
| `DomainId` | identité d'un nom de domaine | `SCONE-DOMAIN-V1` | voir `/docs/fr/general/naming.md` |
| `RecordHash` | identité du contenu DNS | `SCONE-RECORD-V1` | voir `/docs/fr/technical/protocol.md` |
| `TxId` | identité d'une transaction | `SCONE-TX-V1` | ci-dessous |
| `BlockHash` | identité d'un bloc | `SCONE-BLOCK-V1` | ci-dessous |

L'arbre de Merkle utilise son propre préfixe `SCONE-MERKLE-V1`. Ces
préfixes s'ajoutent à ceux de `scone-core` (`SCONE-PUBKEY-V1`,
`SCONE-OWNER-V1`).

Chaîne de dépendances complète :

```text
DomainId
    ↓
RegisterDomain / UpdateDomain
    ↓
Transaction
    ↓
TxId
    ↓
MerkleRoot
    ↓
BlockHeader
    ↓
BlockHash
    ↓
prev_hash of the next block
    ↓
canonical chain
```

## TxId

```text
TxId = BLAKE3-256("SCONE-TX-V1" || canonical_encode(Transaction))
```

- déterministe : la même transaction (même encodage canonique) ⇒ même
  `TxId`, sur tous les nœuds, pour toujours ;
- identité **par contenu** : tout champ modifié change le `TxId`
  (domaine, propriétaire, horodatage, preuve, séquence, record_hash) ;
- aucun horodatage local ni valeur non canonique n'entre dans le calcul ;
- aucune dépendance au stockage ou au réseau ;
- l'identité ne garantit pas l'unicité d'application : rejouer une
  transaction identique échoue sur les règles d'état (double
  `RegisterDomain`, rejeu de séquence), pas sur le `TxId`.

API : `scone_blockchain::transaction_id(&Transaction) -> Result<TxId>`.

## Arbre de Merkle (tx_root)

Engagement sur la **liste ordonnée** des transactions d'un bloc :

```text
0 transactions  root = BLAKE3-256("SCONE-MERKLE-V1" || "EMPTY")
leaf            BLAKE3-256("SCONE-MERKLE-V1" || 0x00 || TxId)
internal node   BLAKE3-256("SCONE-MERKLE-V1" || 0x01 || left || right)
odd level       the last hash is duplicated to complete the pair
```

Règles :

- **jamais trié** : l'ordre des transactions dans un bloc est
  significatif ; deux blocs contenant les mêmes transactions dans un
  ordre différent ont des racines différentes ;
- les étiquettes `0x00` / `0x01` empêchent toute confusion entre une
  feuille et un nœud interne (seconde préimage) ;
- le préfixe `SCONE-MERKLE-V1` isole l'arbre des autres domaines de
  hachage (un `TxId` ne peut pas être interprété comme un nœud) ;
- implémentation maison (~30 lignes), déterministe et testée ; pas de
  bibliothèque Merkle externe.

API : `merkle_root(&[TxId]) -> MerkleRoot` et
`tx_root(&[Transaction]) -> Result<MerkleRoot>`.

## BlockHash

```text
BlockHash = BLAKE3-256("SCONE-BLOCK-V1" || canonical_encode(BlockHeader))
```

Le hachage couvre **uniquement** l'en-tête canonique : les
transactions sont déjà engagées par `tx_root`, elles ne sont donc pas
hachées une seconde fois :

```text
transactions -> TxIds -> MerkleRoot -> BlockHeader -> BlockHash
```

Modifier une transaction, leur ordre, la hauteur, l'horodatage, le
`prev_hash` ou la charge de consensus change le `BlockHash`.

API : `block_hash(&BlockHeader) -> Result<BlockHash>`.

## Réseaux et genèse (M8b)

Scone définit trois réseaux disjoints, identifiés par un `network_id`
canonique (1..=16 octets ASCII `[a-z0-9-]`) :

| Réseau | `network_id` | apex TLD privé | apex TLD ouvert | apex domaine | plancher TLD | plancher domaine |
|---|---|---|---|---|---|---|
| testnet | `scone-testnet` | 8 bits | 7 bits | 6 bits | 2 bits | 1 bit |
| mainnet | `scone-mainnet` | 28 bits | 27 bits | 26 bits | 25 bits | 22 bits |
| devnet | `scone-devnet` | 2 bits | 1 bit | 1 bit | 1 bit | 1 bit |

Les paramètres vivent dans `scone-core::network::NetworkParams`
(instances nommées `TESTNET` / `MAINNET` / `DEVNET`, recherche via
`NetworkParams::by_name`). Modifier une difficulté = un changement de
consensus **pour ce réseau**.

**Invariant structurel** : la devnet est une troisième `NetworkParams`
complète — jamais un sous-ensemble d'options de la testnet. La testnet
reste **strictement identique** à ce qu'elle était avant l'ajout de la
devnet (mêmes constantes Argon2id, mêmes courbes, mêmes fenêtres
d'époque) ; la suite de tests épingle cet invariant.

**Difficulté variable par longueur** (`scone-core::pow_length`) : la
difficulté d'une création dépend de la longueur du **label créé** (le
TLD pour `RegisterTld` — privé ou nouvellement ouvert —, le label le
plus à gauche pour `RegisterDomain`). La formule donne :

```text
TLD     : difficulty(len) = apex − round(0.287 × (len − 1)), clamped at the floor
domain  : difficulty(len) = apex − round(0.480 × (len − 1)), clamped at the floor
```

Deux pentes distinctes : `log₂(1.22) ≈ 0.287` bit/caractère pour les
TLD (chaque caractère supplémentaire divise le coût par ~1.22), et
`log₂(1.40) ≈ 0.480` bit/caractère pour les domaines — les domaines
décroissent plus vite : leur valeur réside dans le TLD qui les porte.
Un TLD né ouvert se situe à l'apex privé − 1 ; un domaine à l'apex
privé − 2 (mainnet : TLD privé 25, TLD ouvert 24, domaine 23).
Benchmarks mainnet (Argon2id à mémoire dure, ~11.4 H/s mono-thread) :
TLD privé 1 caractère = 28 bits ≈ 30 jours, 9+ caractères = plancher
25 bits ≈ 3.8 jours ; TLD ouvert 1 caractère = 27 bits ≈ 15 jours ;
domaine 1 caractère = 26 bits ≈ 3.8 jours, 9+ caractères = plancher
22 bits ≈ 11.5 h. **Machine de référence 8 cœurs** (recalibrage
2026-10 — le CLI mine multi-cœurs), ~11.4 H/s/cœur. **L'ouverture**
d'un TLD privé existant (`SetTldOpen`) reste à difficulté plate
(`open_tld_pow_difficulty` : 6 bits testnet / 22 bits mainnet) : la
transaction ne porte que le `tld_id`, le nom a payé sa longueur à la
création. Fonction pure (déterminisme : deux nœuds calculent la même
difficulté pour le même nom).

La genèse est une constante de protocole **par réseau**, reproductible
à partir des seules constantes (pas d'aléa, pas d'horloge locale, pas
de réseau) :

| Champ | Valeur |
|---|---|
| `version` | `PROTOCOL_VERSION` (`1`) |
| `height` | `0` |
| `prev_hash` | 32 octets à zéro (pas de bloc précédent) |
| `tx_root` | racine de Merkle de la liste vide |
| `timestamp` | `GENESIS_TIMESTAMP = 0` (purement structurel) |
| `consensus` | les octets canoniques du `network_id` |
| transactions | vide |

Variante **genèse d'allocation** (kickstarter mainnet, M-next) : le
champ `consensus` porte `network_id ‖ ALLOC-V1 ‖ count.v ‖ entries`
(domaines/TLD prépayés, tri canonique par nom, manifeste ≤ 60 KiB,
≤ 2048 allocations) — ce format élargi est ce qui justifie
`MAX_CONSENSUS_LEN` = 64 KiB (voir `scone-blockchain::genesis_alloc`).

Le `network_id` dans la charge de consensus rend les hachages de
genèse — et donc toute la blockchain — **disjoints entre les
réseaux** : un bloc testnet ne peut jamais s'attacher à une chaîne
mainnet (`UnknownParent`), aucun rejeu n'est possible sans tout
re-signer.

API : `genesis_of(&NetworkParams) -> Block`,
`genesis_hash_of(NetworkId) -> BlockHash` ; les alias `genesis()` /
`genesis_hash()` désignent la testnet (valeur par défaut en
développement).

### Devnet — réseau local réinitialisable

La devnet (`scone-devnet`) est le troisième réseau, dédié à
l'itération locale rapide :

- **Genèse propre** : l'identifiant de réseau `scone-devnet` dans la
  charge de consensus produit un hachage de genèse distinct de la
  testnet ET de la mainnet — aucune genèse, bloc, signature ou preuve
  PoW ne franchit jamais la frontière (le rejeu est impossible).
- **PoW quasi nul** : courbe épinglée dans la plage 0..=2 bits
  (`DEVNET_APEX = 2`, `DEVNET_FLOOR = 1` — la décroissance par
  longueur s'applique toujours) : une réclamation de TLD ou de domaine
  se mine en millisecondes, en exerçant exactement le même chemin de
  code que les autres réseaux.
- **Consensus PoS rapide** : mêmes paramètres que la testnet (comité
  de 4, époques de 10 s) pour une finalité rapide sur un maillage
  local.
- **Réinitialisation** : la chaîne vit dans son propre répertoire de
  données (`~/.scone/devnet/` par défaut). **Effacer ce répertoire et
  relancer le relais réinitialise la chaîne** (genèse fraîche, store
  vierge) — aucun outil supplémentaire nécessaire. Ne JAMAIS pointer
  un répertoire de données devnet vers celui d'un autre réseau : un
  store non vide contenant une autre chaîne est refusé (`Corrupted`).

## État canonique

```text
DomainState  = { owner: OwnerId, sequence: u64, record_hash: Option<RecordHash>,
                 registered_at: u64, valid_until: u64 }
TldState     = enum { Private { owner: OwnerId } | Open }
               -- enum (owner-option): the open/private invariant is
               -- structural; the death deadline lives outside the entry
               -- (tld_floors table + journaled death schedule)
ChainState   = NetworkParams                  (bound network, M8b)
               + DomainId -> DomainState      (direct access, in memory)
               + TldId -> TldState            (TLD registry, M7b)
               + DomainId -> (expired_at, OwnerId)   (grace windows, M8b)
```

* `registered_at` / `valid_until` (M8b) : expiration de
  l'enregistrement, évaluée par rapport à l'**horodatage du bloc
  parent** (déterministe, engagé dans l'en-tête — jamais l'horloge
  locale) ;
* `open` : un TLD privé peut **naître ouvert** (`RegisterTld` avec
  `open = 0x01` — à personne, libre-service) ou **fermé** à la
  réclamation (fermé = attribution uniquement), puis devenir ouvert via
  `SetTldOpen` (auto-enregistrement avec PoW) — **irréversiblement** ;

- accès direct par `DomainId` (32 octets) : jamais une `String` comme
  clé, jamais un parcours complet — une condition nécessaire pour
  viser des centaines de milliards de domaines ;
- `record_hash == None` tant qu'aucun `UpdateDomain` n'a été appliqué ;
- les espaces d'identifiants des TLD et des domaines sont **disjoints
  par construction** (préfixes de dérivation distincts, voir
  `/docs/fr/general/naming.md`) : enregistrer le TLD `uip` ne peut
  jamais squatter l'identité d'un domaine, ni l'inverse.

### Règles d'application

`state.apply(transaction)` est déterministe et atomique (en cas
d'erreur, l'état est inchangé) :

**Toutes les transactions (M8b, wire 2026-09)** : les transactions ne
portent plus de champ `network` — la séparation repose sur la
**signature** : la charge signée injecte le `network_id` de la chaîne
(`validate_transaction(tx, network_of_the_chain)`), ainsi une
transaction signée pour un autre réseau échoue à la vérification de
signature (`InvalidSignature`, AVANT application — une tx signée
testnet n'est jamais appliquée, jamais mise en mempool par un nœud
mainnet, et inversement).

**REGISTER_TLD** :

- le TLD doit être **libre** (absent du registre des TLD) et **non
  réservé ICANN** (`IcannTldReserved` : les TLD racine ICANN
  n'appartiennent à personne sur Scone) ;
- **PoW obligatoire** : `pow::verify` sur le défi
  `SCONE-TLD-V1 || tld` (dérivé du nom porté par la tx), à la
  difficulté **fondée sur la longueur** du label (`pow_length`) :
  naissance privée → `tld_difficulty` (apex réseau), naissance ouverte
  (`open = 0x01`) → `open_tld_difficulty` (apex − 2) — le condensé est
  recalculé from scratch
  (sel `SCONE-POW-V1-ARGON2 ‖ network ‖ challenge ‖ epoch`, cœur
  Argon2id), la
  difficulté auto-déclarée doit être égale à la difficulté attendue, et
  l'**époque déclarée** par la preuve doit être égale à
  `height/POW_WINDOW` du bloc appliquant (sel d'époque contre le
  pré-minage — rejet typé `ProofEpochMismatch`, voir
  `transactions.md`) ;
- à l'application : `open = 0x01` → `{ Open }` (à personne —
  libre-service, siège PoS libéré, plancher de mort posé à
  `now + grace`) ; `open = 0x00` → `{ Private { owner } }` — **fermé
  par défaut** (attribution uniquement) ; l'ouverture est explicite
  via `SetTldOpen`.

**REGISTER (RegisterDomain)** :

- **le TLD du nom porté doit être enregistré ET ouvert** : la chaîne
  dérive elle-même `TldId(name.tld())` (jamais une valeur fournie) ;
  absent → `UnknownTld` (D1, M7c), présent mais fermé → `TldClosed`
  (M8b : le chemin attribution uniquement est exclusif) ;
- le domaine doit être **libre** : absent de l'état ET hors de toute
  fenêtre de grâce d'un enregistrement expiré (voir le GC ci-dessous) ;
- **PoW obligatoire** : défi `SCONE-DOMAIN-V1 || name` à la
  **difficulté fondée sur la longueur du label le plus à gauche**
  (`pow_length::domain_difficulty` — le TLD étant ouvert, c'est le
  prix d'entrée en libre-service) ;
- à l'application : `{ owner, sequence: 0, record_hash: None,
  registered_at: now, valid_until: now + 1 year }` —
  **l'enregistrement dure 1 an** (365 jours).

**Ordre intra-bloc** : significatif. `[RegisterTld, SetTldOpen,
RegisterDomain]` dans le même bloc est valide ; toute permutation qui
applique une règle avant que sa précondition existe est rejetée comme
bloc entier (journal d'annulation).

**UPDATE (UpdateDomain)** :

- le domaine doit exister (non expiré) ;
- `tx.owner` doit être le propriétaire actuel ;
- `tx.sequence` doit être **exactement** `current_sequence + 1`
  (ni trou, ni rejeu) ;
- à l'application : `sequence` avance, `record_hash` est remplacé.

**TRANSFER_TLD (0x47)** :

- le TLD doit exister et être **privé** (`UnknownTld` / `TldIsOpen`
  sinon — un TLD ouvert n'appartient à personne) ;
- le signataire doit être le propriétaire actuel (`NotTldOwner`
  sinon) ;
- à l'application : `owner = new_owner` (l'identité du destinataire
  est opaque ; le lien vers les clés pouvant le dépenser se fait par
  les tx suivantes). Pas de séquence : le premier transfert appliqué
  gagne (ordre de la chaîne).

**REVOKE_TLD (0x6B)** :

- le TLD doit exister et être **privé** (un TLD ouvert ne peut pas
  être révoqué — `TldIsOpen` : il meurt naturellement au GC),
  signataire = propriétaire actuel ;
- à l'application : le TLD est retiré du registre et redevient
  réclamable par un `RegisterTld` frais (PoW compris).

**SET_TLD_OPEN (0xB8)** :

- le TLD doit exister et être **privé** (déjà ouvert → `TldIsOpen` :
  l'ancien propriétaire est effacé, plus aucune clé légitime
  n'existe pour signer) ; `open = 0x00` (fermeture) → rejet typé
  `TldCloseUnsupported` dès la validation pure : **l'ouverture est
  irréversible** (owner-option) ;
- signataire = propriétaire actuel (`NotTldOwner`) ;
- **ouverture = PoW obligatoire** : `pow::verify` sur le défi
  `SCONE-TLD-OPEN-V1 || tld_id` à la difficulté plate du réseau
  `open_tld_pow_difficulty` (6 bits testnet / 22 bits mainnet), sel
  d'époque inclus (la preuve voyage dans le champ signé `open_proof`
  de la tx) ;
- à l'application : `TldState::Open` — **le propriétaire est
  effacé** (irréversible, siège PoS libéré), un plancher de mort est
  posé à `now + grace`.

**ASSIGN_DOMAIN (0xD4)** :

- le TLD du nom porté doit exister et être **privé** (`TldIsOpen`
  sinon : un TLD ouvert est libre-service uniquement), signataire =
  propriétaire du TLD (`NotTldOwner`) ;
- le domaine doit être libre (même règle de grâce que REGISTER) ;
- à l'application : `{ owner: assignee, sequence: 0, record_hash:
  None, registered_at: now, valid_until: now + 1 year }`. **Pas de
  PoW** : c'est le propriétaire de l'espace de noms qui se porte
  garant.

**TRANSFER_DOMAIN (0xE2, M8c)** :

- le domaine doit exister (`UnknownDomain` sinon), signataire =
  propriétaire actuel (`NotOwner`) ;
- à l'application : `owner = new_owner` (le siège PoS suit le domaine
  dans le pool du propriétaire, symétrique de `TransferTld`). Pas de
  séquence : le premier transfert appliqué gagne (ordre de la
  chaîne).

**RENEW_DOMAIN (0x3C)** :

- le domaine doit exister ; **le type du TLD est évalué EN PREMIER**
  (durcissement M8c) :
  - **TLD privé** : autorisation **PoS** — le signataire EST le
    propriétaire du TLD, ou le siège du propriétaire du TLD délègue
    vers la clé publique du signataire
    (`effective_operator(tld_owner) == renew.public_key` — la
    délégation est toujours lue DEPUIS le propriétaire du TLD, jamais
    l'inverse). Le champ `proof` doit être vide (une preuve non vide =
    rejet).
  - **TLD ouvert** : autorisation **PoW** — le signataire est le
    propriétaire actuel du domaine (`NotOwner` sinon) et la preuve
    mine le défi de renouvellement : même difficulté que la création
    (par longueur de label), **liée à l'expiration actuelle**
    (`challenge = DOMAIN_ID_VERSION ‖ canonical name ‖ current
    valid_until`) — la preuve d'un renouvellement est distincte de
    celle de l'enregistrement et de chaque renouvellement antérieur
    (anti-réutilisation) ;
- `valid_until` doit **étendre strictement** l'échéance actuelle
  (`RenewalNotExtending` sinon) ;
- `valid_until ≤ now + 3 years` (`RenewalExceedsTerm` — borne
  anti-thésaurisation : **1 terme de renouvellement = 1 an, 3 ans
  d'avance maximum**) ;
- à l'application : `valid_until = tx.valid_until` ; le renouvellement
  repousse aussi le plancher de mort du TLD ouvert sous-jacent (la tx
  porte le nom canonique à cette fin).

**GC déterministe (M8b)** : avant d'appliquer les transactions d'un
bloc, la chaîne évalue les expirations à l'**horodatage du bloc
parent** : tout domaine dont `valid_until ≤ now` est retiré de l'état
vivant et parqué dans une **fenêtre de grâce de 30 jours**
(`valid_until + 30 days`). Durant la période de grâce, seul l'ancien
propriétaire peut réclamer le nom ; ensuite, il est libre pour tous.
Le GC est journalisé : un bloc rejeté restaure l'état pré-GC bit par
bit.

**Cycle de vie des TLD (termes fixes)** : les TLD portent leur propre
horloge, indépendante des domaines.

- **TLD privé** : la réclamation ouvre un **terme de 3 ans**
  (`TLD_TERM_SECS`). Le renouvellement (`RenewTld`, 0x8F) paie un
  **PoW égal à la création** et réinitialise exactement 3 ans (terme
  fixe redérivé à l'application). À l'échéance, une **grâce de 60
  jours** (`TLD_GRACE_SECS`) s'ouvre : le renouvellement reste
  possible, mais **toute autre opération de gestion est gelée** —
  notamment `AssignDomain` (rejet typé `TldExpiredFrozen`) : un
  propriétaire expiré perd ses droits de gestion jusqu'au
  renouvellement. Les domaines déjà attribués en dessous vivent leur
  propre `valid_until` — le TLD gèle, pas ses enfants. Grâce écoulée
  sans renouvellement : le TLD meurt au GC (siège PoS libéré,
  réclamable). Un `TransferTld` porte le terme restant.
- **TLD ouvert** : né (ou ouvert) avec un **plancher d'1 an**
  (`OPEN_TLD_FLOOR_SECS` — anti création-destruction instantanée).
  Chaque domaine vivant en dessous repousse l'horloge de mort (elle
  suit l'expiration du domaine le plus récent). Plus aucun domaine +
  plancher écoulé : mort au GC, réclamable.
- **TLD ouvert immortel** (genèse uniquement : `.scone`, `.tor` —
  nommage vérifiable au-dessus d'adresses onion via CNAME —, `.p2p`) :
  l'allocation `OpenImmortalTld` pose la sentinelle `until =
  u64::MAX` — le GC ne peut jamais la franchir et le réarmement
  monotone ne peut jamais l'abaisser : le TLD n'expire jamais, même
  vidé de tous ses domaines. Aucune transaction d'exécution ne peut
  créer un immortel (`RegisterTld` garde son plancher d'1 an) ; les
  domaines en dessous vivent leur cycle normal.

Les morts de TLD partagent la file de mort déterministe des domaines
(`tld_deaths`), journalisée et restaurable bit à bit.

**Exception de hauteur 1** : la genèse porte un horodatage
STRUCTTURELLEMENT nul (`GENESIS_TIMESTAMP` = 0, une convention de
protocole — pas une horloge). Le temps-chaîne du PREMIER bloc est donc
son propre horodatage (engagé dans son en-tête, déterministe), pas
celui de son parent : ancrer les expirations du bloc 1 à 0 épinglerait
le plancher de mort d'un TLD né ouvert (et le `valid_until` d'un
domaine né au bloc 1) à l'époque Unix — le GC pré-application du bloc
2 (horodatage réel) retirerait la ressource avant application
(« dropped conflicting tx from block production: unknown TLD »,
régression testnet 60).

**Rejeu** : le rejeu inter-réseaux est impossible à trois niveaux —
généses disjointes (`UnknownParent`), `network_id` injecté dans toute
charge de signature (signature invalide sur l'autre réseau — le seul
endroit où le réseau apparaît depuis le wire 2026-09), PoW séparé par
réseau dans le condensé. Le rejeu intra-réseau d'une tx identique
échoue sur les règles d'état (double réclamation, séquence), jamais
sur le `TxId`.

Deux nœuds du même réseau partant du même état et appliquant les mêmes
blocs dans le même ordre produisent exactement le même état final.

## Ordre des transactions

L'ordre est une propriété fondamentale, engagée à chaque niveau :

```text
transaction order in the block
    ↓
MerkleRoot
    ↓
BlockHash
    ↓
canonical chain
```

Deux transactions ne sont pas interchangeables : dans le même bloc,
`[REGISTER, UPDATE]` s'applique, `[UPDATE, REGISTER]` est rejeté
(domaine inexistant au moment du `UPDATE`). Le départage global (qui
ordonne le mempool, qui produit le bloc) appartient au consensus.

## Validation des blocs (push_block)

Tout est **recalculé**, rien n'est pris sur foi — un `tx_root`, un
hachage, une signature ou un champ fourni par un pair n'est jamais
pris pour argent comptant :

1. `prev_hash` == hachage du sommet canonique (sinon `UnknownParent`
   si le parent est inconnu, `ParentNotTip` s'il est connu mais pas le
   sommet : détection minimale de fourche) ;
2. `height` == hauteur du sommet + 1 ;
3. `version` == `PROTOCOL_VERSION` **exactement** (toute autre
   version — inférieure ou supérieure — est rejetée, pas
   réinterprétée) ;
4. nombre de transactions ≤ `MAX_TXS_PER_BLOCK` (4096) ;
5. `tx_root` recalculé sur les transactions dans l'ordre du bloc ;
6. hooks de consensus (`validate_header`, `validate_tx`) ;
6ter. **anti-rejeu TXID fenêtré (port .bak)** : une transaction dont
   le `TxId` apparaît dans l'index d'inclusion des
   `REPLAY_WINDOW_BLOCKS = 256` blocs les plus récents est rejetée
   (`TxReplay`). L'index (TxId → hauteur d'inclusion) est une pure
   fonction de la chaîne canonique, élagué au même rythme ; l'index
   est reconstruit au démarrage par le stockage (rebalayage des
   `min(256, height)` derniers blocs persistés — M8, voir
   `/docs/fr/technical/storage.md`) : un nœud redémarré rejette les
   mêmes rejeux qu'un nœud vivant. Au-delà de la fenêtre, la
   ré-inclusion reste soumise aux règles d'état (double réclamation,
   rejeu de séquence).
6bis. **producteur signé (M5, port .bak)** : la charge `consensus`
   doit se décoder comme une charge de producteur
   (`SCONE-BLOCK-V1 ‖ pk[32] ‖ sig[64]`, bornée) ; la signature
   couvre le *hachage de signature du producteur* = le `BlockHash` de
   l'en-tête **avec le champ `consensus` vidé** (la charge vit dans ce
   champ ; signer le hachage complet serait circulaire) ; le
   producteur doit appartenir à l'ensemble autorisé à l'`horodatage`
   du bloc (comité élu par point de contrôle, plus tirages de
   récupération déverrouillés ; amorçage : propriétaires des domaines
   vivants — un pool vide = production ouverte, sinon la première
   inscription serait impossible). La genèse (hauteur 0, charge vide)
   est structurelle et jamais concernée. Erreur typée
   `InvalidProducer` ;
7. pour chaque transaction, dans l'ordre : validation
   **cryptographique** (`validate_transaction` : lien
   propriétaire/clé recalculé + `verify_strict` sur la charge signée
   recalculée from scratch, voir
   `/docs/fr/technical/transactions.md`), puis application à l'état
   canonique sous un **journal d'annulation** (undo-log : une entrée
   par transaction appliquée, coût O(transactions du bloc), pas
   O(domaines)) ;
8. commit atomique : tout passe ⇒ bloc ajouté, journal abandonné ; la
   moindre erreur ⇒ rembobinage du journal (ordre inverse) et la
   chaîne rigoureusement inchangée, bit pour bit.

Aucune fonction de validation ne panique ; toute entrée malformée ou
hostile produit une `BlockchainError` typée
(`OwnerKeyMismatch`, `InvalidSignature`, …).

## Assemblage des blocs (BlockBuilder)

`BlockBuilder` (`scone-blockchain::builder`) assemble un bloc à partir
d'une file de transactions validées :

```text
BlockBuilder::after(parent_height, prev_hash)
    .with_timestamp(t) .with_consensus(payload)
    .push_tx(tx)… .build() -> Block
```

- `tx_root` **recalculé** sur la liste ordonnée des transactions en
  file (jamais fourni) ;
- `prev_hash`/`height` chaînés sur le parent (le hachage du parent est
  calculé par l'appelant via `block_hash`, jamais pris d'un pair) ;
- `version` = `PROTOCOL_VERSION` ;
- la borne `MAX_TXS_PER_BLOCK` appliquée à l'insertion
  (`TooManyTransactions`) ;
- la charge `consensus` (charge de producteur signé) et l'`horodatage`
  sont fournis par l'appelant : le consensus concret se branche via le
  trait [`Consensus`](#abstraction-de-consensus) existant,
  symétriquement à la validation ;
- M5 : `.with_producer(&SigningKey)` signe le bloc comme producteur —
  la charge `consensus` devient la charge de producteur signée (sur le
  hachage de signature du producteur défini ci-dessus) et remplace
  tout `.with_consensus` antérieur.

Un bloc assemblé passe `push_block` sans ré-encodage (testé).

## Fourches et choix de fourche (M3c, port .bak)

`push_block` reste le point d'entrée linéaire (parent == sommet,
sinon `UnknownParent`/`ParentNotTip`). `try_attach` (M3c) évalue les
blocs concurrents :

- parent inconnu → `UnknownParent` (le bloc attend) ;
- lignée valide mais en retard → évaluation de branche : la chaîne la
  plus longue gagne ; égalité parfaite → départage déterministe
  (plus petit hachage de sommet dans l'ordre lexicographique — deux
  nœuds détenant les mêmes blocs choisissent toujours le même
  gagnant, testé) ;
- **plancher de finalité** : une branche dont un ancêtre contredit un
  point de contrôle finalisé (inclus dans le `prev_checkpoint_hash`
  d'un point de contrôle accepté) est refusée (erreur typée
  `Consensus("branch contradicts a finalized checkpoint")`) ;
- adoption = rejeu depuis la genèse des blocs de la branche gagnante
  (déterminisme bit à bit testé : deux nœuds recevant les mêmes blocs
  dans des ordres différents aboutissent au même état) ;
- aucun bloc reçu n'est canonique avant validation complète.

## Fenêtre RAM bornée (M7a, port .bak)

La chaîne ne garde en RAM que les **512 derniers blocs**
(`RAM_WINDOW_BLOCKS`, port du `HISTORY_KEEP = 512` du .bak), au-dessus
du **plancher de finalité** (le dernier point de contrôle finalisé) :

- un bloc ne quitte la RAM que s'il est **à la fois** hors de la
  fenêtre (`height < tip − 511`) **et** strictement sous le plancher
  de finalité — une réorg au-dessus du plancher reste toujours
  possible, ces blocs ne sont donc JAMAIS évincés ;
- avant le premier point de contrôle finalisé (amorçage), **rien
  n'est évincé** : la mémoire bornée commence avec la finalité ;
- l'éviction est réelle : blocs, index de hachage (`known_hashes`,
  lui-même borné, indexé par hauteur) et txs quittent la mémoire — le
  store (`NodeStore`, `scone-storage`) détient les copies durables ;
- `block(h)` sous la fenêtre → `None` ; `block_result(h)` → erreur
  typée `BlockPruned { height }` — les chemins de réorg (`push_block`,
  `try_attach`, `adopt_branch`, rejeu de préfixe) la propagent : le
  relais doit recharger le segment depuis le stockage ou retomber sur
  une synchronisation complète (limitation documentée : pas
  d'adoption segmentée avant le port complet du relais) ;
- un bloc construit sur un parent évincé se classe `BlockPruned` (pas
  `UnknownParent`) : hauteur du parent < `base_height` ⇒ le store peut
  le servir.

## Abstraction de consensus

```rust
pub trait Consensus {
    fn validate_header(&self, header: &BlockHeader) -> Result<()>;
    fn validate_tx(&self, tx: &Transaction) -> Result<()>;
}
```

- `PermissiveConsensus` (défaut) : accepte tout — utilisé jusqu'à la
  définition des règles réelles ;
- les hooks sont appelés par `push_block` : le consensus concret
  (PoW, difficulté, frais, horodatage/anti-rejeu, choix de fourche)
  se branchera sans réécrire la chaîne ;
- une seule abstraction, pas de hiérarchie de traits.

## État passage à l'échelle (backend KV, pivot SMT, file d'expiration, pool de propriétaires borné)

Objectif : 250+ milliards de domaines potentiels — ne JAMAIS tout
charger en RAM. Quatre pièces :

### Backend KV (`state_backend.rs`)

```rust
pub trait StateBackend: Default + Send {
    fn get(&self, key: &Pkey) -> Option<Vec<u8>>;
    fn put(&mut self, key: Pkey, value: Vec<u8>);
    fn delete(&mut self, key: &Pkey);
    fn range(&self, from: Option<&Pkey>, limit: usize)
        -> (Vec<(Pkey, Vec<u8>)>, Option<Pkey>);
}
```

- `Pkey` = les 32 octets bruts du `DomainId`/`TldId` — UN espace de
  clés ordonné (préfixes de dérivation disjoints), valeurs =
  encodages `SCONE-ENTRY-DOM-V1` (88 octets fixes) /
  `SCONE-ENTRY-TLD-V1` (variable : 33 octets privé / 1 octet ouvert,
  enum owner-option — drapeau + propriétaire conditionnel, plus de
  champ `until`) ;
- `MemoryBackend` (BTreeMap) = implémentation de référence (dev,
  tests) ; AUCUNE dépendance concrète dans `scone-blockchain` (redb
  interdit dans la crate) — protocole/consensus dépendent du TRAIT
  seul ;
- `CachedBackend<B>` (`state_cache.rs`) = cache LRU borné au-dessus
  d'un backend interne : write-through (l'interne est toujours à
  jour), les manques ne sont jamais mis en cache, `range` page
  l'interne — mêmes réponses que le backend interne seul (contrat de
  déterminisme, testé) ;
- `RedbBackend` (`scone-storage::redb_backend`) = backend persistant :
  table `state_kv` (clés brutes de 32 octets, ordre lexicographique =
  ordre croissant requis), compteur maintenu dans la table méta
  `state_kv_meta` (jamais COUNT) ; `CachedBackend<RedbBackend>` = état
  à empreinte RAM bornée — le « palier léger » ;
- `smt_persist` : consomme les ensembles sales du SMT
  (`take_dirty_sets`) et les écrit dans un backend sous des clés à
  préfixe réservé (0x01 nœud, 0x02 feuille) — fondation du SMT paginé ;
- le trait n'exige plus `Default` (une base de données disque n'a pas
  de valeur par défaut sensée) ;
- `range` itère par ordre de clé croissant (contrat de déterminisme) —
  chemin archive uniquement, jamais une règle de consensus ;
- `ChainState` accède aux domaines/TLD UNIQUEMENT par clé à travers le
  backend (lectures ponctuelles, écritures journalisées).

### Racine SMT canonique (pivot)

- `state_root_smt()` (O(1), racine cachée, `SCONE-STATE-SMT-V1`) est
  le state_root **canonique** : c'est ce que `checkpoint_data` engage
  et ce que `accept_checkpoint` vérifie au sommet. Le calcul ne
  traverse JAMAIS l'ensemble des domaines ;
- l'ancien pliage direct `SCONE-STATE-V1` survit comme
  `state_root_archive()` : O(N), chemin archive/vérification
  (re-pliage complet depuis le stockage pour contre-vérifier une
  racine historique) — pas un artefact de consensus. Les deux
  engagent les mêmes feuilles : états logiquement égaux ⇒ racines
  égales (pour les deux) ;
- un point de contrôle accepté avec la racine SMT se revérifie après
  une restauration (reload) : la racine est une fonction pure de
  l'état restauré (backend + SMT + index).

### File d'expiration (GC O(expirés), pas O(N))

- `expirations: BTreeMap<u64, BTreeSet<DomainId>>` — instant
  d'expiration → ids expirant alors (ids triés par créneau : égalité
  d'état et ordre de vidange indépendants de l'ordre d'arrivée) ;
- `gc_expired_journaled(now)` vide `range(..=now)` au lieu de
  balayer tout l'état : coût O(expirés à cet instant) ;
- borne RAM : une entrée par domaine VIVANT (expirations en attente
  uniquement — un renouvellement re-clé le créneau, le GC le vide),
  jamais un nœud par domaine dans une structure de balayage ;
- maintenu par le journal d'annulation (rembobinage bit à bit : le
  créneau restauré est remis en place).

### Pool de propriétaires borné (des clés, pas des domaines)

- `owner_refcounts: HashMap<OwnerId, u64>` — un compteur par
  propriétaire vivant DISTINCT (domaines + TLD), maintenu par le
  journal à chaque register/transfer/revoke/assign/GC
  (acquire/release) ;
- `eligible_validators(now)` lit cet index — O(propriétaires),
  JAMAIS une itération sur les domaines. La vivacité temporelle est
  déjà garantie : le GC déterministe (horodatage parent) a retiré
  tout domaine expiré ET son compteur de référence avant l'appel.
  Sortie triée (déterministe) ;
- borne RAM : O(propriétaires distincts), indépendant du nombre de
  domaines ;
- `owner_pool_root` (pliage archive) plie ces propriétaires triés —
  aucun balayage de domaines non plus.

Budget RAM en régime établi (backend disque/réseau) :
O(propriétaires distincts + expirations en attente + fenêtres de
grâce en cours + journal d'annulation) — tout borné indépendamment du
nombre de domaines, tout restauré bit à bit par le journal.

## Sièges et comité de finalité (mécanisme PoS exact)

Cette section décrit le mécanisme EXACT d'éligibilité et d'élection du
comité tel qu'implémenté (`finality.rs`) — le complément opérationnel
(boucle d'ancrage, wire) vit dans `/docs/fr/technical/relay.md`,
l'analyse de sécurité dans `/docs/fr/technical/security-model.md`.

### Un siège = une possession (pas d'enregistrement)

Un **siège** PoS n'est PAS un enregistrement : c'est la possession
d'un domaine ou d'un TLD vivant. `eligible_validators(now)` calcule le
pool :

- clés des propriétaires vivants via l'index borné `owner_refcounts`
  (un propriétaire DISTINCT par domaine/TLD vivant — cent domaines
  sous une clé = UN siège : l'anti-Sybil économique est par
  identité, linéaire) ;
- un TLD compte (sa réclamation a coûté un PoW — le propriétaire d'un
  TLD vivant est un parties prenante), un domaine vivant aussi
  (`valid_until > now` — garanti par le GC déterministe, déjà vidé
  avant l'appel) ;
- **moins les clés bannies** (M9 : un équivoqueur slashé est hors du
  pool à vie) ;
- un TLD **ouvert** (`owner: None`) ne détient AUCUN siège : son
  ancien propriétaire est libéré à l'ouverture (`owner_release` dans
  l'apply de `SetTldOpen`), et il meurt au GC sans jamais revoter.

Sortie triée (ordre déterministe, indépendant de l'ordre d'itération
de l'index).

Conséquence structurelle : **la mort du siège = la mort du
domaine/TLD**. Expiration + GC, `RevokeTld` (siège libéré),
`TransferTld`/`TransferDomain` (le siège suit le domaine vers le
nouveau propriétaire), ouverture `SetTldOpen` (siège libéré) — chaque
transition de propriété déplace ou libère le siège dans le pool,
symétriquement au journal `owner_refcounts` (acquire/release).

### Pool GELÉ au dernier point de contrôle finalisé (`FinalizedBase.pool`)

À l'acceptation d'un point de contrôle (`accept_checkpoint`), la base
de finalité capture le pool de l'état AU BLOC FINALISÉ :

```text
FinalizedBase { checkpoint, pool, block_ts, seed }
```

- `pool` = `eligible_validators(horodatage du bloc finalisé)` — la
  finalité ne relit JAMAIS l'état vivant ensuite : toute inscription
  (réclamation, transfert, bannissement) après le point de contrôle
  ne compte que pour les époques ultérieures ;
- `seed` = `next_seed(signing_hash(checkpoint))` — la graine de
  l'époque SUIVANTE dérive du hachage de **contenu** du point de
  contrôle (keyed, ensemble de signatures exclu — deux vues détenant
  des sous-ensembles de quorum distincts du même point de contrôle
  élisent le même comité).

### Tirage du comité (amorçage, fraîcheur, dégradation)

`committee(recovery)` :

1. **Amorçage** (pas de point de contrôle finalisé) : le pool ENTIER
   signe — un comité tiré d'une vue locale divergerait entre nœuds en
   retard (la cause racine des gels de finalité) ; le pool entier est
   déterministe par la chaîne. La sélection stricte reprend à la
   première finalité.
2. **Régime finalisé** : tirage dans le sous-ensemble **frais** du
   pool gelé (RFC proof-of-service §5.2) — chaque siège (clé
   propriétaire) est résolu vers son opérateur EFFECTIF par l'état au
   sommet (délégation `SetTldOperator`/`SetDomainOperator` = un fait
   on-chain récent, ou la clé propriétaire en auto-exploitation) ; le
   siège participe SOUS SA CLÉ EFFECTIVE (c'est celle qui signe), à
   condition d'avoir un **battement de cœur frais** (fenêtre glissante).
   Les clés bannies restent exclues.
3. **Dégradation sûre** : moins de `MIN_FINALITY_COMMITTEE_SIZE` (4)
   opérateurs frais → tirage dans le pool ENTIER (amorçage à froid,
   panne générale des opérateurs — la finalité n'est jamais perdue par
   construction).

`allowed_producers(block_ts)` (production de blocs) suit le même
schéma : pool entier à l'amorçage ; comité élu + tirages de
récupération (tirage k déverrouillé après k × `epoch_secs` de temps de
consensus) en régime finalisé.

Le quorum de point de contrôle est `quorum_for(taille du comité)`
(floor(2n/3)+1) ; un comité sous `MIN_FINALITY_COMMITTEE_SIZE` ne
finalise jamais (garde BFT).

## Différé au consensus

Délibérément non défini dans cette crate :

- **mempool** : admission/réordonnancement fin (frais, remplacement)
  reste dans le relais (M4 a apporté le cap par domaine +
  l'anti-rejeu TXID fenêtré — `REPLAY_WINDOW_BLOCKS = 256`) ;
- **ordre global** : le tri du mempool est un tri local déterministe
  (relais), pas une autorité globale ;
- **autorité d'horodatage** au-delà du GC : les expirations (M8b)
  utilisent l'horodatage du bloc parent ; les autres règles
  temporelles (anti-rejeu fin) restent au consensus ;
- stockage distribué : l'état est en mémoire ; le backend
  (`scone-storage`, redb) applique ces règles sans les dupliquer.

Livré depuis M8b (auparavant différé) : PoW d'enregistrement par
réseau (`RegisterTld` toujours, `RegisterDomain` sur un TLD ouvert),
expiration/renouvellement 1 an + grâce 30 jours + horizon 3 ans, GC
déterministe à l'horodatage du bloc parent, rejeu inter-réseaux
impossible (genèse + champ signé + condensé PoW).

## Simulation distribuée déterministe (P1.6)

`crates/scone-blockchain/src/sim/` contient un harnais de simulation
réseau **déterministe** (`SimNet`) : N `Blockchain`s réelles (aucun
mock) pilotées par une **horloge virtuelle** et une **file d'événements**
priorisée par `(tick, ordre d'insertion)`. Pas de sockets, pas
d'async, pas d'horloge murale — chaque tirage (perte, latence,
duplication) est une fonction pure de `(seed, numéro d'envoi)`, ainsi
un scénario à graine fixe est reproductible octet pour octet.

### Modèle

- **Transport** : chaque émission subit une perte probabiliste
  (configurable en ‰), une latence artificielle bornée
  (`latency_ticks`), une duplication possible (une copie retardée) et
  la carte de partition courante (les messages inter-groupes sont
  perdus). Les duplications n'avancent jamais l'horloge logique et ne
  déclenchent aucun nouvel envoi : le flux d'événements réel est
  identique avec ou sans duplication (contrat de déterminisme, prouvé
  par le test `duplicate_and_delay`).
- **Gossip épidémique** : un nœud retransmet à ses pairs connectés
  (même partition) un bloc/transaction/point de contrôle uniquement à
  première vue — règle de relais H2 (coupe les boucles). Un bloc
  parqué (orphe) n'est JAMAIS rediffusé ; les lacunes de réception
  sont comblées par le chemin de synchronisation (`GetBlocks`),
  comme le relais.
- **Production** : un pas de production global par intervalle ; chaque
  groupe de partition produit au PLUS UN bloc par tour, par son nœud
  autorisé détenant la meilleure hauteur. Durant l'amorçage (avant la
  première finalité), le producteur est épinglé au nœud le plus bas :
  l'ensemble de producteurs autorisés dérive de l'état (pool des
  domaines vivants), deux vues légèrement différentes élisent des
  comités différents et un bloc localement légitime serait
  `InvalidProducer` ailleurs — une divergence permanente. Après la
  première finalité, le comité est gelé dans la base de finalité
  (identique sur tout nœud convergé) et la production est libre.
- **Finalité** : à chaque multiple de `sign_interval` hauteurs,
  l'ancre désignée du comité signe le contenu déterministe de son
  sommet via un `SignerGuard` réel résistant au crash (une clé = un
  vote par époque, état fsyncé avant diffusion) ; les autres membres
  du comité signent à réception (après avoir vérifié que le bloc
  référencé est bien leur bloc canonique à cette hauteur — garde au
  niveau relais) et rediffusent l'agrégat enrichi. Au quorum,
  `accept_checkpoint` finalise. Chaque nœud s'engage sur la première
  proposition vue par époque (garde d'amorçage anti-double-finalité :
  deux propositions concurrentes de la même époque à signataires
  disjoints ne peuvent pas s'accumuler toutes deux).
- **Crash/redémarrage** : un nœud crashé perd tout son état RAM
  (chaîne, mempool, parking, vues) mais garde son répertoire
  signer-guard sur disque, exactement comme un vrai processus ; il se
  resynchronise en rejouant les blocs canoniques servis par ses pairs
  (`GetBlocks`/`GetCheckpoints`).

### Scénarios couverts (`sim::tests`)

| Test | Ce qui est prouvé |
|---|---|
| `convergence_simple` | 8 nœuds, 5 ‰ de perte, latence 1–5 ticks, 500 blocs : même sommet, même `state_root_smt` partout ; finalité active ; jamais deux points de contrôle incompatibles par époque. |
| `partition_then_heal` | partition 4a/4b pendant ~300 blocs : les deux côtés produisent, guérison → convergence avec réorg de la branche perdante ; les fenêtres de points de contrôle croisées ne contredisent jamais une époque partagée (avant et après guérison). |
| `crash_restart` | crash à h≈200, redémarrage à h≈400 : resynchronisation par rejeu des blocs du réseau — même hauteur, sommet, état ; la transaction post-crash est visible du nœud ressuscité. |
| `duplicate_and_delay` | 10 ‰ de duplication : hauteur finale, racine d'état et nombre de points de contrôle finalisés identiques au run sans duplication (idempotence : TxReplay/seen-sets absorbent les doublons). |
| `concurrent_tx` | deux propriétaires concurrents sur le même domaine : exactement une mise à jour gagne (le propriétaire — `NotOwner` pour l'intrus, rejet typé propre), jamais double application d'état, la tx perdante n'entre dans aucune chaîne. |

### Paramètres et bornes de durée

Valeurs par défaut : `produce_interval` 7 ticks, `sign_interval` 16
hauteurs, `sync_interval` 29–60 ticks (41 par défaut, abaissé dans
certains tests), `horizon` 200k–500k ticks virtuels. Chaque test
intègre une garde de temps réel < 60 s en mode debug
(`assert_under_60s`). Le coût dominant est le noyau de validation
(~9 ms par `push_block_with_gc` vide en debug, Ed25519 non optimisé) :
les scénarios restent sous ~8 nœuds × 500 blocs.

### Limites

- **Pas de vraie couche de transport** : ce harnais exerce la logique
  de chaîne sous livraison adverse, pas l'essaim (libp2p,
  request-response, backpressure, découverte). Les vrais tests e2e
  (`crates/scone-network/tests/`) restent la preuve de transport.
- **Synchronisation par lots en RAM** : les blocs servis viennent de
  la fenêtre RAM (les scénarios restent sous `RAM_WINDOW_BLOCKS`) ; le
  chemin `BlockPruned` + rechargement depuis le store n'est pas
  simulé (couvert par les tests unitaires de la chaîne).
- **Modèle d'ancre honnête** : les ancres ne signent que du contenu
  vérifiable contre leur propre chaîne canonique ; l'équivoque
  byzantine (double signature délibérée) est empêchée par le
  `SignerGuard` mais pas exercée comme attaque active.
- **Une partition à la fois** : la carte de groupes est globale et
  modifiée séquentiellement par le pilote ; les partitions qui se
  chevauchent/évoluent en continu ne sont pas modélisées.

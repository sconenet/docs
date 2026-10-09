# Blockchain

Document de référence de la couche blockchain : identités, hash, genèse,
état canonique, ordre des transactions et validation des blocs.
L'implémentation de référence est la crate
[`scone-blockchain`](../../../crates/scone-blockchain), au-dessus de
`scone-protocol` (encodage canonique), `scone-core` (règles métier) et
`scone-crypto` (BLAKE3).

Périmètre actuel : logique blockchain avec consensus réel (producteur
signé, checkpoints, finalité), **état en mémoire** au-dessus d'un
backend KV (le stockage concret vit dans `scone-storage`) — voir
[Différé au consensus](#différé-au-consensus).

## Identités et séparation des domaines de hash

Quatre identités, quatre rôles, quatre préfixes BLAKE3 distincts — un
préfixe n'est jamais réutilisé entre deux usages :

| Type | Rôle | Préfixe | Formule |
|---|---|---|---|
| `DomainId` | identité du nom de domaine | `SCONE-DOMAIN-V1` | voir `/docs/fr/general/naming.md` |
| `RecordHash` | identité du contenu DNS | `SCONE-RECORD-V1` | voir `/docs/fr/technical/protocol.md` |
| `TxId` | identité de la transaction | `SCONE-TX-V1` | ci-dessous |
| `BlockHash` | identité du bloc | `SCONE-BLOCK-V1` | ci-dessous |

Le Merkle tree utilise son propre préfixe `SCONE-MERKLE-V1`. À ces
préfixes s'ajoutent ceux de `scone-core` (`SCONE-PUBKEY-V1`,
`SCONE-OWNER-V1`).

Chaîne de dépendance complète :

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
prev_hash du bloc suivant
    ↓
chaîne canonique
```

## TxId

```text
TxId = BLAKE3-256("SCONE-TX-V1" || canonical_encode(Transaction))
```

- déterministe : même transaction (même encodage canonique) ⇒ même
  `TxId`, sur tous les nœuds, pour toujours ;
- identité **par contenu** : tout champ modifié change le `TxId`
  (domaine, owner, timestamp, proof, sequence, record_hash) ;
- aucun timestamp local ni valeur non canonique n'entre dans le calcul ;
- aucune dépendance au stockage ou au réseau ;
- l'identité ne garantit pas l'unicité d'application : rejouer une
  transaction identique échoue sur les règles d'état (double
  `RegisterDomain`, replay de séquence), pas sur le `TxId`.

API : `scone_blockchain::transaction_id(&Transaction) -> Result<TxId>`.

## Merkle tree (tx_root)

Engagement sur la **liste ordonnée** des transactions d'un bloc :

```text
0 transaction   root = BLAKE3-256("SCONE-MERKLE-V1" || "EMPTY")
feuille         BLAKE3-256("SCONE-MERKLE-V1" || 0x00 || TxId)
nœud interne    BLAKE3-256("SCONE-MERKLE-V1" || 0x01 || left || right)
niveau impair   le dernier hash est dupliqué pour compléter la paire
```

Règles :

- **jamais de tri** : l'ordre des transactions d'un bloc est
  significatif ; deux blocs contenant les mêmes transactions dans un
  ordre différent ont des racines différentes ;
- les tags `0x00` / `0x01` empêchent toute confusion entre feuille et
  nœud interne (second-preimage) ;
- le préfixe `SCONE-MERKLE-V1` isole l'arbre des autres domaines de
  hash (un `TxId` ne peut pas être interprété comme un nœud) ;
- implémentation maison (~30 lignes), déterministe et testée ; aucune
  bibliothèque Merkle externe.

API : `merkle_root(&[TxId]) -> MerkleRoot` et
`tx_root(&[Transaction]) -> Result<MerkleRoot>`.

## BlockHash

```text
BlockHash = BLAKE3-256("SCONE-BLOCK-V1" || canonical_encode(BlockHeader))
```

Le hash ne couvre **que** le header canonique : les transactions y sont
déjà engagées par `tx_root`, elles ne sont donc pas hashées une seconde
fois :

```text
transactions -> TxIds -> MerkleRoot -> BlockHeader -> BlockHash
```

Modifier une transaction, leur ordre, la hauteur, le timestamp, le
`prev_hash` ou le payload consensus change le `BlockHash`.

API : `block_hash(&BlockHeader) -> Result<BlockHash>`.

## Réseaux et genèse (M8b)

Scone définit trois réseaux disjoints, identifiés par un
`network_id` canonique (1..=16 octets ASCII `[a-z0-9-]`) :

| Réseau | `network_id` | apex TLD privé | apex TLD ouvert | apex domaine | plancher TLD | plancher domaine |
|---|---|---|---|---|---|---|
| testnet | `scone-testnet` | 8 bits | 7 bits | 6 bits | 2 bits | 1 bit |
| mainnet | `scone-mainnet` | 25 bits | 24 bits | 23 bits | 22 bits | 19 bits |
| devnet | `scone-devnet` | 2 bits | 1 bit | 1 bit | 1 bit | 1 bit |

```text
TLD     : difficulté(len) = apex − round(0,287 × (len − 1)), bornée au plancher
domaine : difficulté(len) = apex − round(0,480 × (len − 1)), bornée au plancher
```

Deux pentes distinctes : `log₂(1,22) ≈ 0,287` bit/caractère pour les
TLD (chaque caractère supplémentaire divise le coût par ~1,22), et
`log₂(1,40) ≈ 0,480` bit/caractère pour les domaines — les domaines
décroissent plus vite : leur valeur est dans le TLD qui les porte.
Le TLD né ouvert vaut l'apex privé − 1 ; le domaine vaut l'apex privé
− 2 (mainnet : TLD privé 25, TLD ouvert 24, domaine 23). Repères
mainnet (Argon2id à mémoire dure, ~11,4 H/s mono-cœur) : TLD privé
1 car. = 25 bits ≈ 34 j, 9+ car. = plancher 22 bits ≈ 4,3 j ;
TLD ouvert 1 car. = 24 bits ≈ 17 j ; domaine 1 car. = 23 bits ≈ 8,5 j,
9+ car. = plancher 19 bits ≈ 13 h. L'**ouverture** d'un TLD privé
existant (`SetTldOpen`) reste à difficulté plate
(`open_tld_pow_difficulty` : 6 bits testnet / 22 bits mainnet) : la tx
ne porte que le `tld_id`, le nom a payé sa longueur à la création.
Fonction pure (déterminisme : deux nœuds calculent la même difficulté
pour le même nom).

La genèse est une constante de protocole **par réseau**, reproductible
à partir des constantes seules (aucun aléa, aucune horloge locale,
aucun réseau) :

| Champ | Valeur |
|---|---|
| `version` | `PROTOCOL_VERSION` (`1`) |
| `height` | `0` |
| `prev_hash` | 32 octets nuls (aucun bloc précédent) |
| `tx_root` | Merkle root de la liste vide |
| `timestamp` | `GENESIS_TIMESTAMP = 0` (purement structurel) |
| `consensus` | les octets canoniques du `network_id` |
| transactions | vide |

Variante **genèse d'allocation** (kickstarter mainnet, M-next) : le
champ `consensus` porte `network_id ‖ ALLOC-V1 ‖ count.v ‖ entrées`
(domaines/TLDs pré-payés, tri canonique par nom, manifeste ≤ 60 KiB,
≤ 2048 allocations) — c'est ce format élargi qui justifie
`MAX_CONSENSUS_LEN` = 64 KiB (voir `scone-blockchain::genesis_alloc`).

Le `network_id` dans le payload consensus rend les hashs de genèse —
et donc toute la chaîne de blocs — **disjoints entre réseaux** : un
bloc testnet ne peut jamais s'attacher à une chaîne mainnet
(`UnknownParent`), aucun replay n'est possible sans tout re-signer.

API : `genesis_of(&NetworkParams) -> Block`,
`genesis_hash_of(NetworkId) -> BlockHash` ; les alias `genesis()` /
`genesis_hash()` désignent la testnet (défaut de développement).

### Devnet — réseau local réinitialisable

La devnet (`scone-devnet`) est le troisième réseau, dédié à
l'itération locale rapide :

- **Genèse propre** : le `network_id` `scone-devnet` dans le payload
  consensus produit un hash de genèse distinct de la testnet ET de la
  mainnet — aucune genèse, aucun bloc, aucune signature, aucune
  preuve PoW ne franchit la frontière (replay impossible).
- **PoW quasi nul** : courbe épinglée dans 0..=2 bits
  (`DEVNET_APEX = 2`, `DEVNET_FLOOR = 1` — la décroissance par
  longueur s'applique quand même) : un claim TLD ou domaine se mine
  en quelques millisecondes, pour exercer exactement le même chemin
  de code que les autres réseaux.
- **Consensus PoS rapide** : mêmes paramètres que la testnet (comité
  de 4, époques de 10 s) pour une finalité rapide en maillage local.
- **Réinitialisation** : la chaîne vit dans son data-dir dédié
  (`~/.scone/devnet/` par défaut). **Effacer ce répertoire puis
  relancer le relay réinitialise la chaîne** (genèse fraîche, store
  vierge) — aucun outil supplémentaire n'est nécessaire. Ne JAMAIS
  pointer un data-dir devnet vers un data-dir d'un autre réseau : un
  store non vide qui tient une autre chaîne est refusé (`Corrupted`).

## État canonique

```text
DomainState  = { owner: OwnerId, sequence: u64, record_hash: Option<RecordHash>,
                 registered_at: u64, valid_until: u64 }
TldState     = enum { Private { owner: OwnerId } | Open }
               -- énum (owner-option) : l'invariant ouvert/privé est
               -- structurel ; l'échéance de mort vit hors de l'entrée
               -- (table tld_floors + horaire de morts journalisé)
ChainState   = NetworkParams                  (réseau lié, M8b)
               + DomainId -> DomainState      (accès direct, en mémoire)
               + TldId -> TldState            (registre TLD, M7b)
               + DomainId -> (expired_at, OwnerId)   (fenêtres de grâce, M8b)
```

* `registered_at` / `valid_until` (M8b) : l'expiration de
  l'enregistrement, évaluée contre le **timestamp du bloc parent**
  (déterministe, engagé dans le header — jamais l'horloge locale) ;
* `open` : un TLD privé peut naître **ouvert** (`RegisterTld` avec
  `open = 0x01` — personne, self-service) ou **fermé** à la claim
  (fermé = assign-only), puis devenir ouvert via `SetTldOpen`
  (auto-enregistrement avec PoW) — **irréversiblement** ;

- accès direct par `DomainId` (32 octets) : jamais de `String` comme
  clé, jamais de scan complet — condition nécessaire pour viser des
  centaines de milliards de domaines ;
- `record_hash == None` tant qu'aucun `UpdateDomain` n'a été appliqué ;
- les espaces d'ids TLD et domaine sont **disjoints par construction**
  (préfixes de dérivation distincts, voir `/docs/fr/general/naming.md`) :
  enregistrer le TLD `uip` ne peut jamais squatter l'identité d'un
  domaine, ni l'inverse.

### Règles d'application

`state.apply(transaction)` est déterministe et atomique (en cas
d'erreur, l'état est inchangé) :

**Toutes les transactions (M8b, wire 2026-09)** : les transactions ne
portent plus de champ `network` — la séparation repose sur la
**signature** : le payload signé injecte le `network_id` de la chaîne
(`validate_transaction(tx, network_de_la_chaîne)`), donc une tx
signée pour un autre réseau échoue à la vérification de signature
(`InvalidSignature`, AVANT l'application — une tx testnet-signée
n'est jamais appliquée, jamais poolée par un nœud mainnet, et
inversement).

**REGISTER_TLD** :

- le TLD doit être **libre** (absent du registre TLD) et **non
  réservé ICANN** (`IcannTldReserved` : les TLD racine ICANN
  n'appartiennent à personne sur Scone) ;
- **PoW obligatoire** : `pow::verify` sur le challenge
  `SCONE-TLD-V1 || tld` (dérivé du nom porté par la tx), à la
  difficulté **par longueur** du label (`pow_length`) : naissance
  privée → `tld_difficulty` (apex réseau), naissance ouverte
  (`open = 0x01`) → `open_tld_difficulty` (apex − 2) — le digest est
  recalculé de zéro
  (sel `SCONE-POW-V1-ARGON2 ‖ network ‖ challenge ‖ epoch`, cœur
  Argon2id), la
  difficulté auto-déclarée doit égaler la difficulté attendue, et
  l'**époque déclarée** par la preuve doit égaler
  `height/POW_WINDOW` du bloc appliquant (sel d'époque anti
  pré-minage — rejet typé `ProofEpochMismatch`, voir
  `transactions.md`) ;
- à l'application : `open = 0x01` → `{ Open }` (personne —
  self-service, siège PoS libéré, plancher de mort posé à
  `now + grâce`) ; `open = 0x00` → `{ Private { owner } }` —
  **fermé par défaut** (assign-only) ; l'ouverture est explicite via
  `SetTldOpen`.

**REGISTER (RegisterDomain)** :

- **le TLD du nom porté doit être enregistré ET ouvert** : la chaîne
  dérive elle-même `TldId(nom.tld())` (jamais une valeur fournie) ;
  absent → `UnknownTld` (D1, M7c), présent mais fermé → `TldClosed`
  (M8b : le chemin assign-only est exclusif) ;
- le domaine doit être **libre** : absent de l'état ET hors de toute
  fenêtre de grâce d'un enregistrement expiré (voir GC ci-dessous) ;
- **PoW obligatoire** : challenge `SCONE-DOMAIN-V1 || nom` à la
  difficulté **par longueur du label de gauche**
  (`pow_length::domain_difficulty` — le TLD étant ouvert, c'est le
  prix d'entrée auto-service) ;
- à l'application : `{ owner, sequence: 0, record_hash: None,
  registered_at: now, valid_until: now + 1 an }` — **l'enregistrement
  dure 1 an** (365 jours).

**Ordre intra-bloc** : significatif. `[RegisterTld, SetTldOpen,
RegisterDomain]` dans un même bloc est valide ; toute permutation qui
applique une règle avant que sa précondition existe est rejetée en
bloc entier (undo-log).

**UPDATE (UpdateDomain)** :

- le domaine doit exister (non expiré) ;
- `tx.owner` doit être le propriétaire actuel ;
- `tx.sequence` doit être **exactement** `sequence_courante + 1`
  (aucun trou, aucun replay) ;
- à l'application : `sequence` avance, `record_hash` est remplacé.

**TRANSFER_TLD (0x47)** :

- le TLD doit exister et être **privé** (`UnknownTld` /
  `TldIsOpen` sinon — un TLD ouvert n'appartient à personne) ;
- le signataire doit être l'owner courant (`NotTldOwner` sinon) ;
- à l'application : `owner = new_owner` (l'identité du destinataire
  est opaque ; le binding aux clés qui peuvent la dépenser se fait
  par les tx suivantes). Pas de séquence : premier transfert appliqué
  gagne (ordre de chaîne).

**REVOKE_TLD (0x6B)** :

- le TLD doit exister et être **privé** (un TLD ouvert ne peut pas
  être révoqué — `TldIsOpen` : il meurt naturellement au GC),
  signataire = owner courant ;
- à l'application : le TLD est retiré du registre et re-claimable par
  un `RegisterTld` frais (PoW compris).

**SET_TLD_OPEN (0xB8)** :

- le TLD doit exister et être **privé** (déjà ouvert → `TldIsOpen` :
  l'ancien owner est effacé, il n'existe plus de clé légitime pour
  signer) ; `open = 0x00` (fermeture) → rejet typé
  `TldCloseUnsupported` dès la validation pure : **l'ouverture est
  irréversible** (owner-option) ;
- signataire = owner courant (`NotTldOwner`) ;
- **ouverture = PoW obligatoire** : `pow::verify` sur le challenge
  `SCONE-TLD-OPEN-V1 || tld_id` à la difficulté plate
  `open_tld_pow_difficulty` du réseau (6 bits testnet / 22 bits
  mainnet), sel d'époque compris (la preuve voyage dans le champ
  signé `open_proof` de la tx) ;
- à l'application : `TldState::Open` — **l'owner est effacé**
  (irréversible, siège PoS libéré), un plancher de mort est posé à
  `now + grâce`.

**ASSIGN_DOMAIN (0xD4)** :

- le TLD du nom porté doit exister et être **privé**
  (`TldIsOpen` sinon : un TLD ouvert est self-service uniquement),
  signataire = owner du TLD (`NotTldOwner`) ;
- le domaine doit être libre (même règle de grâce que REGISTER) ;
- à l'application : `{ owner: assignee, sequence: 0, record_hash:
  None, registered_at: now, valid_until: now + 1 an }`. **Pas de
  PoW** : c'est l'owner du namespace qui se porte garant.

**TRANSFER_DOMAIN (0xE2, M8c)** :

- le domaine doit exister (`UnknownDomain` sinon), signataire = owner
  courant (`NotOwner`) ;
- à l'application : `owner = new_owner` (le siège PoS suit le domaine
  dans le pool d'owners, symétrique de `TransferTld`). Pas de
  séquence : premier transfert appliqué gagne (ordre de chaîne).

**RENEW_DOMAIN (0x3C)** :

- le domaine doit exister ; **le type du TLD est évalué EN PREMIER**
  (M8c, durcissement) :
  - **TLD privé** : autorisation **PoS** — le signataire EST
    l'owner du TLD, ou le siège de l'owner du TLD délègue vers la
    clé publique du signataire
    (`effective_operator(tld_owner) == renew.public_key` — la
    délégation se lit TOUJOURS depuis le TLD owner, jamais
    l'inverse). Le champ `proof` doit être vide (une preuve
    non vide = rejet).
  - **TLD ouvert** : autorisation **PoW** — le signataire est
    l'owner du domaine courant (`NotOwner` sinon) et la preuve
    mine le challenge de renouvellement : même difficulté que la
    création (par longueur de label), **liée à l'échéance
    courante** (`challenge = DOMAIN_ID_VERSION ‖ nom canonique ‖
    valid_until actuel`) — la preuve d'un renouvellement est
    distincte de celle de l'enregistrement et de tout
    renouvellement antérieur (anti-réutilisation) ;
- `valid_until` doit **strictement étendre** l'échéance courante
  (`RenewalNotExtending` sinon) ;
- `valid_until ≤ now + 3 ans` (`RenewalExceedsTerm` — borne
  anti-thésaurisation : **1 terme de renouvellement = 1 an,
  3 ans d'avance maximum**) ;
- à l'application : `valid_until = tx.valid_until` ; le renouvellement
  reporte aussi le plancher de mort du TLD ouvert sous-jacent (la tx
  porte le nom canonique pour cela).

**GC déterministe (M8b)** : avant l'application des transactions
d'un bloc, la chaîne évalue les expirations au **timestamp du bloc
parent** : tout domaine dont `valid_until ≤ now` est retiré de l'état
vivant et parqué dans une **fenêtre de grâce de 30 jours**
(`valid_until + 30 j`). Pendant la grâce, seul l'ancien owner peut
re-réclamer le nom ; après, il est libre pour tous. Le GC est
journalisé : un bloc rejeté restaure l'état pré-GC bit à bit.

**Cycle de vie des TLD (terme fixe)** : les TLD ont leur propre
horloge, indépendante des domaines.

- **TLD privé** : le claim ouvre un **terme de 3 ans**
  (`TLD_TERM_SECS`). Le renouvellement (`RenewTld`, 0x8F) paie une
  **PoW égale à la création** et remet exactement 3 ans (terme
  fixe re-dérivé à l'application). À l'échéance s'ouvre une
  **grâce de 60 jours** (`TLD_GRACE_SECS`) : le renouvellement y
  reste possible, mais **toute autre gestion est gelée** —
  notamment l'`AssignDomain` (rejet typé `TldExpiredFrozen`) : un
  owner expiré perd son droit de gestion jusqu'à renouvellement.
  Les domaines déjà attribués dessous vivent leur propre
  `valid_until` — c'est le TLD qui gèle, pas ses enfants. Grâce
  écoulée sans renouvellement : le TLD meurt au GC (siège PoS
  libéré, re-claimable). Un `TransferTld` porte le terme restant.
- **TLD ouvert** : né (ou ouvert) avec un **plancher de 1 an**
  (`OPEN_TLD_FLOOR_SECS` — anti création-destruction instantanée).
  Chaque domaine vivant dessous repousse l'horloge de mort (elle
  suit l'échéance du domaine le plus récent). Plus aucun domaine
  + plancher écoulé : mort au GC, re-claimable.
- **TLD ouvert immortel** (genèse uniquement : `.scone`,
  `.tor` — naming vérifiable au-dessus des adresses onion via
  CNAME —, `.p2p`) :
  l'allocation `OpenImmortalTld` pose la sentinelle
  `until = u64::MAX` — le GC ne peut jamais la franchir et le
  re-arm monotone ne peut jamais la baisser : le TLD n'expire
  jamais, même vidé de ses domaines. Aucune transaction runtime
  ne peut créer d'immortel (`RegisterTld` garde son plancher
  d'1 an) ; les domaines dessous vivent leur cycle normal.

Les morts de TLD partagent la même file de mort déterministe que
les domaines (`tld_deaths`), journalisées et restaurables bit à
bit.

**Exception hauteur 1** : la genèse porte un timestamp
STRUCTURELLEMENT nul (`GENESIS_TIMESTAMP` = 0, convention
protocolaire — pas une horloge). Le temps chaîne du PREMIER bloc est
donc son propre timestamp (engagé dans son header, déterministe),
et non celui du parent : ancrer les expirations du bloc 1 sur 0
fixerait le plancher de mort d'un TLD né ouvert (et le
`valid_until` d'un domaine né au bloc 1) à l'époque Unix — le GC
pré-apply du bloc 2 (timestamp réel) retirerait la ressource avant
l'application (« dropped conflicting tx from block production:
unknown TLD », régression testnet 60).

**Replay** : le replay cross-réseau est impossible à trois niveaux —
genesis disjointes (`UnknownParent`), `network_id` injecté dans
chaque payload de signature (signature invalide sur l'autre réseau —
seule place du network depuis le wire 2026-09), PoW séparé par réseau
dans le digest. Le replay intra-réseau d'une
tx identique échoue sur les règles d'état (double claim, séquence),
jamais sur le `TxId`.

Deux nœuds du même réseau partant du même état et appliquant les
mêmes blocs dans le même ordre produisent exactement le même état
final.

## Ordre des transactions

L'ordre est une propriété fondamentale, engagée à chaque niveau :

```text
ordre des transactions dans le bloc
    ↓
MerkleRoot
    ↓
BlockHash
    ↓
chaîne canonique
```

Deux transactions ne sont pas interchangeables : dans un même bloc,
`[REGISTER, UPDATE]` s'applique, `[UPDATE, REGISTER]` est rejeté
(domaine inexistant au moment de l'`UPDATE`). Le tie-break global
(qui ordonne le mempool, qui produit le bloc) appartient au consensus.

## Validation d'un bloc (push_block)

Tout est **recalculé**, rien n'est cru sur parole — un `tx_root`, un
hash, une signature ou un champ fourni par un pair n'est jamais pris
pour argent comptant :

1. `prev_hash` == hash de la pointe canonique (sinon `UnknownParent`
   si le parent est inconnu, `ParentNotTip` s'il est connu mais pas la
   pointe : détection de fork minimale) ;
2. `height` == hauteur pointe + 1 ;
3. `version` == `PROTOCOL_VERSION` **exactement** (toute autre
   version — inférieure ou supérieure — est rejetée, pas
   réinterprétée) ;
4. nombre de transactions ≤ `MAX_TXS_PER_BLOCK` (4096) ;
5. `tx_root` recalculé sur les transactions dans l'ordre du bloc ;
6. crochets du consensus (`validate_header`, `validate_tx`) ;
6ter. **anti-replay TXID fenêtré (port .bak)** : une transaction dont
   le `TxId` figure dans l'index des inclusions de la fenêtre
   `REPLAY_WINDOW_BLOCKS = 256` derniers blocs est rejetée
   (`TxReplay`). L'index (TxId → hauteur d'inclusion) est une
   fonction pure de la chaîne canonique, élaguée du même pas ;
   l'index est reconstruit au boot par le stockage (rescan des
   derniers `min(256, hauteur)` blocs persistés — M8, voir
   `/docs/fr/technical/storage.md`) : un nœud redémarré rejette les
   mêmes rejeux qu'un nœud vivant. Au-delà de la fenêtre, une
   ré-inclusion reste soumise aux règles d'état (double claim,
   replay de séquence).
6bis. **producteur signé (M5, port .bak)** : le payload `consensus`
   doit décoder comme un payload producteur
   (`SCONE-BLOCK-V1 ‖ pk[32] ‖ sig[64]`, borné) ; la signature porte
   sur le *hash de signature producteur* = `BlockHash` du header
   **le champ `consensus` vidé** (le payload vivant dans ce champ,
   signer le hash complet serait circulaire) ; le producteur doit
   appartenir à l'ensemble autorisé à l'instant `timestamp` du bloc
   (comité élu par checkpoint, plus tirages recovery déverrouillés ;
   bootstrap : owners de domaines vivants — pool vide = production
   ouverte, sinon le premier enregistrement serait impossible). La
   genèse (hauteur 0, payload vide) est structurelle et jamais
   concernée. Erreur typée `InvalidProducer` ;
7. pour chaque transaction, dans l'ordre : validation
   **cryptographique** (`validate_transaction` : binding
   owner/clé recomputé + `verify_strict` sur le payload signé
   recomputé de zéro, voir `/docs/fr/technical/transactions.md`), puis
   application à l'état canonique sous **journal d'annulation**
   (undo-log : une entrée par transaction appliquée, coût
   O(transactions du bloc) et non O(domaines)) ;
8. commit atomique : tout passe ⇒ bloc ajouté, journal jeté ; la
   moindre erreur ⇒ rembobinage du journal (ordre inverse) et chaîne
   rigoureusement inchangée, bit à bit.

Aucune fonction de validation ne panique ; toute entrée malformée ou
hostile produit une `BlockchainError` typée
(`OwnerKeyMismatch`, `InvalidSignature`, …).

## Assemblage de blocs (BlockBuilder)

`BlockBuilder` (`scone-blockchain::builder`) assemble un bloc depuis
une queue de transactions validées :

```text
BlockBuilder::after(hauteur_parent, prev_hash)
    .with_timestamp(t) .with_consensus(payload)
    .push_tx(tx)… .build() -> Block
```

- `tx_root` **recalculé** sur la liste ordonnée des transactions
  queued (jamais fourni) ;
- `prev_hash`/`height` chaînés sur le parent (le hash parent est
  calculé par l'appelant via `block_hash`, jamais pris d'un pair) ;
- `version` = `PROTOCOL_VERSION` ;
- borne `MAX_TXS_PER_BLOCK` appliquée à l'ajout
  (`TooManyTransactions`) ;
- le payload `consensus` (payload producteur signé) et le `timestamp` sont
  fournis par l'appelant : le consensus concret s'insère via le trait
  [`Consensus`](#abstraction-du-consensus) existant, symétriquement à
  la validation ;
- M5 : `.with_producer(&SigningKey)` signe le bloc comme producteur —
  le payload `consensus` devient le payload producteur signé (sur le
  hash de signature producteur défini ci-dessus) et remplace tout
  `.with_consensus` préalable.

Un bloc assemblé passe `push_block` sans réencodage (testé).

## Forks et fork choice (M3c, port .bak)

`push_block` reste l'entrée linéaire (parent == pointe, sinon
`UnknownParent`/`ParentNotTip`). `try_attach` (M3c) évalue les blocs
concurrents :

- parent inconnu → `UnknownParent` (le bloc attend) ;
- filiation valide mais en retrait → évaluation de branche : la plus
  longue chaîne gagne ; égalité parfaite → tie-break déterministe
  (hash de pointe le plus petit en ordre lexicographique — deux
  nœuds ayant les mêmes blocs choisissent toujours le même gagnant,
  testé) ;
- **plancher de finalité** : une branche dont un ancêtre contredit un
  checkpoint finalisé (inclus dans `prev_checkpoint_hash` d'un
  checkpoint accepté) est refusée (erreur typée
  `Consensus("branch contradicts a finalized checkpoint")`) ;
- adoption = rejeu depuis la genèse des blocs de la branche gagnante
  (déterminisme bit-exact testé : deux nœuds recevant les mêmes blocs
  dans des ordres différents aboutissent au même état) ;
- aucun bloc reçu n'est canonique avant validation complète.

## Fenêtre RAM bornée (M7a, port .bak)

La chaîne ne garde en RAM que les **512 derniers blocs**
(`RAM_WINDOW_BLOCKS`, port du `HISTORY_KEEP = 512` du .bak), au-dessus
du **plancher de finalité** (le dernier checkpoint finalisé) :

- un bloc quitte la RAM seulement s'il est **à la fois** hors de la
  fenêtre (`height < tip − 511`) **et** strictement sous le plancher
  de finalité — un reorg au-dessus du plancher reste toujours
  possible, ces blocs ne sont donc JAMAIS évincés ;
- avant le premier checkpoint finalisé (bootstrap), **rien n'est
  évincé** : la mémoire bornée commence avec la finalité ;
- l'éviction est réelle : blocs, index de hashs (`known_hashes`,
  borné lui aussi, indexé par hauteur) et txs quittent la mémoire —
  le store (`NodeStore`, `scone-storage`) détient les copies
  durables ;
- `block(h)` sous la fenêtre → `None` ; `block_result(h)` → erreur
  typée `BlockPruned { height }` — les chemins de reorg
  (`push_block`, `try_attach`, `adopt_branch`, rejeu de préfixe) la
  propagent : le relay doit recharger le segment depuis le stockage
  ou retomber sur une sync complète (limitation documentée : pas
  d'adoption segmentée avant le port complet du relay) ;
- un bloc construit sur un parent évincé classe comme `BlockPruned`
  (pas `UnknownParent`) : hauteur du parent < `base_height` ⇒ le
  store peut le servir.

## Abstraction du consensus

```rust
pub trait Consensus {
    fn validate_header(&self, header: &BlockHeader) -> Result<()>;
    fn validate_tx(&self, tx: &Transaction) -> Result<()>;
}
```

- `PermissiveConsensus` (défaut) : accepte tout — utilisé tant que les
  règles réelles ne sont pas définies ;
- les crochets sont appelés par `push_block` : le consensus concret
  (PoW, difficulté, frais, timestamp/anti-replay, fork choice)
  s'insérera sans réécrire la chaîne ;
- une seule abstraction, pas de hiérarchie de traits.

## État scalable (backend KV, pivot SMT, file d'expiration, pool owners borné)

Objectif : 250+ milliards de domaines potentiels — JAMAIS tout charger
en RAM. Quatre pièces :

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
  clés ordonné (préfixes de dérivation disjoints), valeurs = les
  encodages `SCONE-ENTRY-DOM-V1` (88 o fixes) /
  `SCONE-ENTRY-TLD-V1` (variables : 33 o privé / 1 o ouvert,
  owner-option énum — flag + owner conditionnel, plus de champ
  `until`) ;
- `MemoryBackend` (BTreeMap) = implémentation de référence (dev,
  tests) ; AUCUNE dépendance concrète dans `scone-blockchain` (redb
  interdit dans la crate) — le protocole/consensus dépendent du
  TRAIT seul ;
- `CachedBackend<B>` (`state_cache.rs`) = cache LRU borné au-dessus
  d'un backend interne : write-through (l'interne est toujours à
  jour), miss jamais caché, `range` pagine l'interne — mêmes
  réponses que l'interne seul (contrat de déterminisme testé) ;
- `RedbBackend` (`scone-storage::redb_backend`) = backend persisté :
  table `state_kv` (clés 32 o brutes, ordre lexicographique = ordre
  croissant exigé), compteur maintenu en table méta `state_kv_meta`
  (jamais COUNT) ; `CachedBackend<RedbBackend>` = état à empreinte
  RAM bornée — le « tier léger » ;
- `smt_persist` : consomme les ensembles dirty du SMT
  (`take_dirty_sets`) et les écrit dans un backend sous des clés à
  préfixes réservés (0x01 nœud, 0x02 feuille) — fondation du SMT
  paginé ;
- le trait n'exige plus `Default` (une base disque n'a pas de valeur
  par défaut sensée) ;
- `range` itère par ordre de clé croissant (contrat de
  déterminisme) — chemin archive uniquement, jamais une règle de
  consensus ;
- `ChainState` accède aux domaines/TLD UNIQUEMENT par clé via le
  backend (point reads, writes journalisées).

### SMT root canonique (pivot)

- `state_root_smt()` (O(1), racine cachée, `SCONE-STATE-SMT-V1`) est
  le state_root **canonique** : c'est ce que `checkpoint_data`
  commet et ce qu'`accept_checkpoint` vérifie à la pointe. Le
  calcul ne traverse JAMAIS l'ensemble des domaines ;
- l'ancien fold direct `SCONE-STATE-V1` survit comme
  `state_root_archive()` : O(N), chemin d'archive/vérification
  (re-pli complet depuis le stockage pour recouper un root
  historique) — pas un artefact de consensus. Les deux engagent les
  mêmes feuilles : états logiques égaux ⇒ roots égaux (les deux) ;
- un checkpoint accepté avec le root SMT se re-vérifie après un
  restore (reload) : le root est une fonction pure de l'état
  restauré (backend + SMT + index).

### File d'expiration (GC O(expirés), pas O(N))

- `expirations: BTreeMap<u64, BTreeSet<DomainId>>` — instant
  d'expiration → ids expirant alors (ids triés par slot : égalité
  d'état et ordre de drain indépendants de l'ordre d'arrivée) ;
- `gc_expired_journaled(now)` draine `range(..=now)` au lieu de
  scanner l'état entier : coût O(expirés à cet instant) ;
- borne RAM : une entrée par domaine VIVANT (expirations en attente
  uniquement — un renouvellement re-keye le slot, le GC le draine),
  jamais un nœud par domaine dans une structure de scan ;
- maintenu par le journal d'undo (rollback bit-exact : le slot
  restauré est remis).

### Pool d'owners borné (clefs, pas domaines)

- `owner_refcounts: HashMap<OwnerId, u64>` — un compteur par owner
  DISTINCT vivant (domaines + TLDs), maintenu par le journal à
  chaque register/transfer/revoke/assign/GC (acquire/release) ;
- `eligible_validators(now)` lit cet index — O(owners), JAMAIS
  d'itération des domaines. La vivacité temporelle est déjà
  assurée : le GC déterministe (timestamp parent) a retiré tout
  domaine expiré ET son refcount avant l'appel. Sortie triée
  (déterministe) ;
- borne RAM : O(owners distincts), indépendant du nombre de
  domaines ;
- `owner_pool_root` (fold d'archive) plie ces owners triés — pas
  de scan des domaines non plus.

Budget RAM en régime permanent (backend disque/réseau) :
O(owners distincts + expirations en attente + fenêtres de grâce en
vol + journal d'undo) — tout borné indépendamment du nombre de
domaines, tout restauré bit-exact par le journal.

## Sièges et comité de finalité (mécanisme PoS exact)

Cette section décrit le mécanisme EXACT de l'éligibilité et de
l'élection du comité tel qu'implémenté (`finality.rs`) — le
complément opérationnel (boucle anchor, wire) vit dans
`/docs/fr/technical/relay.md`, l'analyse de sécurité dans
`/docs/fr/technical/security-model.md`.

### Siège = possession (aucune inscription)

Un **siège** PoS n'est PAS une inscription : c'est la possession
d'un domaine ou TLD vivant. `eligible_validators(now)` calcule le
pool :

- clés des owners vivants via l'index borné `owner_refcounts`
  (un owner DISTINCT par domaine/TLD vivant — cent domaines pour
  une même clé = UN siège : l'anti-Sybil économique est par
  identité, linéaire) ;
- un TLD compte (son claim a coûté un PoW — l'owner d'un TLD
  vivant est un stakeholder), un domaine vivant
  (`valid_until > now` — garanti par le GC déterministe, déjà
  drainé avant l'appel) ;
- **moins les clés bannies** (M9 : un équivocateur slashé sort du
  pool à vie) ;
- un TLD **ouvert** (`owner: None`) ne porte AUCUN siège : son
  ancien owner est libéré à l'ouverture (`owner_release` dans
  l'apply de `SetTldOpen`), et il meurt au GC sans jamais voter.

Sortie **triée** (ordre déterministe, indépendant de l'ordre
d'itération de l'index).

Conséquence structurelle : **la mort du siège = la mort du
domaine/TLD**. Expiration + GC, `RevokeTld` (siège libéré),
`TransferTld`/`TransferDomain` (le siège suit le domaine vers le
nouveau owner), ouverture `SetTldOpen` (siège libéré) — chaque
transition de l'état d'ownership déplace ou libère le siège dans
le pool, symétriquement au journal `owner_refcounts`
(acquire/release).

### Pool GELÉ au dernier checkpoint finalisé (`FinalizedBase.pool`)

À l'acceptation d'un checkpoint (`accept_checkpoint`), la base de
finalité capture le pool de l'état AU BLOC FINALISÉ :

```text
FinalizedBase { checkpoint, pool, block_ts, seed }
```

- `pool` = `eligible_validators(timestamp du bloc finalisé)` — la
  finalité ne relit JAMAIS l'état vivant ensuite : toute
  inscription (claim, transfert, bannissement) après le checkpoint
  ne compte que pour les epochs suivantes ;
- `seed` = `next_seed(signing_hash(checkpoint))` — la seed de
  l'epoch SUIVANTE dérive du hash de **contenu** du checkpoint
  (keyed, signature-set exclu — deux vues détenant des
  sous-ensembles de quorum distincts du même checkpoint élisent le
  même comité).

### Tirage du comité (bootstrap, frais, dégradation)

`committee(recovery)` :

1. **Bootstrap** (aucun checkpoint finalisé) : TOUT le pool signe —
   un comité tiré d'une vue locale divergerait entre nœuds en
   retard (racine du gel de finalité) ; le pool entier est
   déterministe par la chaîne. La sélection stricte reprend à la
   première finalité.
2. **Régime finalisé** : tirage dans le sous-ensemble **frais** du
   pool gelé (RFC proof-of-service §5.2) — chaque siège (clé
   owner) est résolu en opérateur EFFECTIF par l'état au tip
   (délégation `SetTldOperator`/`SetDomainOperator` = fait
   on-chain récent, ou la clé owner si auto-opéré) ; le siège
   participe SOUS SA CLÉ EFFECTIVE (c'est elle qui signe), à
   condition d'un **heartbeat frais** (fenêtre glissante). Les
   clés bannies restent exclues.
3. **Dégradation sûre** : moins de `MIN_FINALITY_COMMITTEE_SIZE`
   (4) opérateurs frais → tirage dans le pool ENTIER (bootstrap
   froid, panne générale des opérateurs — la finalité n'est jamais
   perdue par construction).

`allowed_producers(block_ts)` (production de blocs) suit le même
schéma : pool entier au bootstrap ; comité élu + tirages recovery
(chaque tirage k déverrouillé après k × `epoch_secs` de temps
consensus) en régime finalisé.

Le quorum de checkpoint est `quorum_for(taille du comité)`
(floor(2n/3)+1) ; un comité sous `MIN_FINALITY_COMMITTEE_SIZE` ne
finalise jamais (garde BFT).

## Différé au consensus

Volontairement non définis dans cette crate :

- **mempool** : l'admission/réordonnancement fin (frais, remplacement)
  reste au relay (M4 a porté cap par domaine + anti-replay TXID
  fenêtré — `REPLAY_WINDOW_BLOCKS = 256`) ;
- **ordering global** : le tri du mempool est un tri déterministe
  local (relay), pas une autorité globale ;
- **timestamp authority** au-delà du GC : les expirations (M8b)
  utilisent le timestamp du bloc parent ; d'autres règles de temps
  (anti-replay fin) restent au consensus ;
- stockage distribué : l'état est en mémoire ; le backend
  (`scone-storage`, redb) applique ces règles sans les dupliquer.

Livré depuis M8b (auparavant différé) : PoW de registration par
réseau (`RegisterTld` toujours, `RegisterDomain` sur TLD ouvert),
expirations/renouvellement 1 an + grâce 30 j + horizon 3 ans, GC
déterministe au timestamp du bloc parent, replay cross-réseau
impossible (genesis + champ signé + digest PoW).

## Simulation distribuée déterministe (P1.6)

`crates/scone-blockchain/src/sim/` contient un harness de simulation
réseau **déterministe** (`SimNet`) : N vraies `Blockchain` (aucun mock)
pilotées par une **horloge virtuelle** et une **file d'événements**
priorisée par `(tick, ordre d'insertion)`. Pas de sockets, pas d'async,
pas d'horloge murale — chaque tirage (perte, latence, duplication) est
une fonction pure de `(seed, numéro d'envoi)`, donc un scénario avec
un seed fixe est reproductible octet par octet.

### Modèle

- **Transport** : chaque émission subit une perte probabiliste
  (paramétrable en ‰), une latence artificielle bornée
  (`latency_ticks`), une duplication éventuelle (une copie retardée)
  et la carte de partitions courante (les messages inter-groupes sont
  perdus). Les duplications n'avancent jamais l'horloge logique et ne
  provoquent aucun envoi nouveau : le flux d'événements réels est
  identique avec ou sans duplication (contrat de déterminisme, prouvé
  par le test `duplicate_and_delay`).
- **Gossip épidémique** : un nœud re-transmet à ses pairs connectés
  (même partition) un bloc/transaction/checkpoint seulement à la
  première vue — règle H2 du relay (coupe les boucles). Un bloc parqué
  (orphan) n'est JAMAIS re-diffusé ; les trous de réception sont
  comblés par la voie de synchronisation (`GetBlocks`), comme le relay.
- **Production** : un pas de production global par intervalle ; chaque
  groupe de partition produit au plus UN bloc par tour, par son nœud
  autorisé détenant la meilleure hauteur. Pendant l'amorçage (avant la
  première finalité) le producteur est épinglé au nœud le plus bas :
  l'ensemble de producteurs autorisés dérive de l'état (pool de
  domaines vivants), deux vues légèrement différentes élisent des
  comités différents et un bloc légitime localement serait
  `InvalidProducer` ailleurs — une divergence permanente. Après la
  première finalité le comité est gelé dans la base de finalité
  (identique sur tout nœud convergé) et la production est libre.
- **Finalité** : à chaque multiple de `sign_interval` hauteurs,
  l'ancre désignée du comité signe le contenu déterministe de son tip
  via un vrai `SignerGuard` crash-safe (une clé = un vote par epoch,
  état fsyncé avant diffusion) ; les autres membres du comité signent
  à la réception (après vérification que le bloc référencé est bien
  leur bloc canonique à cette hauteur — garde de niveau relay) et
  re-diffusent l'agrégat enrichi. Au quorum, `accept_checkpoint`
  finalise. Chaque nœud s'engage sur la première proposition vue par
  epoch (garde anti-double-finalité d'amorçage : deux propositions
  concurrentes de la même epoch avec des signataires disjoints ne
  peuvent pas toutes deux accumuler).
- **Crash/redémarrage** : un nœud crashé perd tout son état RAM
  (chaîne, mempool, parking, vues) mais garde son dossier signer-guard
  sur disque, exactement comme un processus réel ; il resynchronise en
  rejouant les blocs canoniques servis par ses pairs
  (`GetBlocks`/`GetCheckpoints`).

### Scénarios couverts (`sim::tests`)

| Test | Ce qui est prouvé |
|---|---|
| `convergence_simple` | 8 nœuds, 5 ‰ de perte, latence 1–5 ticks, 500 blocs : même tip, même `state_root_smt` partout ; finalité active ; jamais deux checkpoints incompatibles par epoch. |
| `partition_then_heal` | Partition 4a/4b pendant ~300 blocs : les deux côtés produisent, guérison → convergence avec reorg de la branche perdante ; les fenêtres de checkpoints croisées ne contredisent jamais une epoch partagée (avant et après la guérison). |
| `crash_restart` | Crash à h≈200, redémarrage à h≈400 : resynchronisation par rejeu des blocs du réseau — même hauteur, tip, état ; la transaction post-crash est visible du nœud ressuscité. |
| `duplicate_and_delay` | 10 ‰ de duplication : hauteur finale, racine d'état et nombre de checkpoints finalisés identiques au run sans duplication (idempotence : TxReplay/seen-sets absorbent les doublons). |
| `concurrent_tx` | Deux propriétaires concurrents sur le même domaine : exactement une mise à jour gagne (l'owner — `NotOwner` pour l'intrus, rejet typé propre), jamais de double application d'état, la tx perdante n'entre dans aucune chaîne. |

### Paramètres et bornes de durée

Valeurs par défaut : `produce_interval` 7 ticks, `sign_interval` 16
hauteurs, `sync_interval` 29–60 ticks (41 par défaut, abaissé dans
certains tests), `horizon` 200k–500k ticks virtuels. Chaque test
embarque un garde-fou wall-clock < 60 s en mode
debug (`assert_under_60s`). Le coût dominant est le noyau de
validation (~9 ms par `push_block_with_gc` vide en debug, Ed25519 non
optimisé) : les scénarios restent sous ~8 nœuds × 500 blocs.

### Limites

- **Pas de vraie couche transport** : ce harness exerce la logique de
  chaîne sous livraison adverse, pas le swarm (libp2p,
  request-response, backpressure, découverte). Les e2e réels
  (`crates/scone-network/tests/`) restent la preuve transport.
- **Synchronisation par lots en RAM** : les blocs servis viennent de
  la fenêtre RAM (les scénarios restent sous `RAM_WINDOW_BLOCKS`) ; le
  chemin `BlockPruned` + rechargement store n'est pas simulé (couvert
  par les tests unitaires de la chaîne).
- **Modèle d'ancre honnête** : les ancres signent uniquement du
  contenu vérifiable contre leur propre chaîne canonique ; l'équivocation
  byzantine (double-signature délibérée) est empêchée par le
  `SignerGuard` mais pas exercée en tant qu'attaque active.
- **Une partition à la fois** : la carte de groupes est globale et
  changée séquentiellement par le pilote ; les partitions qui se
  recouvrent/évoluent en continu ne sont pas modélisées.

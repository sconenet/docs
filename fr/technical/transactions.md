# Transactions signées (format v1)

Document de référence **normatif** du format des transactions signées.
Il spécifie les champs, l'ordre exact du wire, les tailles, la charge
signée et les règles de validation, de manière à permettre une
implémentation indépendante sans lire le Rust. L'implémentation de
référence est [`scone-protocol`](../../../crates/scone-protocol)
(encodage) et [`scone-blockchain`](../../../crates/scone-blockchain)
(validation).

`PROTOCOL_VERSION` = **1**. Il n'existe qu'un seul jeu de formats : tout
octet de version autre que `0x01` (dans une transaction) ou différent
de la version locale (dans un bloc) est rejeté avec
`UnsupportedVersion`.

## Vue d'ensemble

```text
Transaction = disc u8 || version u8 || payload
```

| Champ | Taille | Rôle |
|---|---|---|
| `disc` | 1 o | type de transaction (tableau ci-dessous) |
| `version` | 1 o | **exactement `0x01`** (version du format tx) |

**Les transactions ne portent plus de champ `network` sur le wire**
(changement 2026-09) : le `network_id` (`str` ≤ 16, ASCII `[a-z0-9-]`
— p. ex. `scone-testnet`, 13 octets) vit **uniquement dans la charge
de signature** — il y est injecté comme premier champ par le
signataire (voir [Charge signée](#charge-signée)) — et reste par
réseau dans le condensé PoW. Une transaction **signée** pour un réseau
ne vérifie donc jamais sur un autre : sa signature échoue
(`InvalidSignature`) contre la charge recalculée par la chaîne, qui
injecte son propre réseau (voir `/docs/fr/technical/blockchain.md` —
réseaux et genèse).

### Discriminants (tableau normatif)

Chaque discriminant est une **constante arbitraire fixe et opaque** :
les valeurs sont délibérément choisies distinctes, afin que le wire ne
suggère **jamais** d'ordre logique entre les types (pas de séquence
0x01/0x02/0x03…). `0x00` n'est **jamais** un type valide (octet de
corruption typique). Les types futurs prendront toute valeur libre
inutilisée, documentée une fois pour toutes dans ce tableau.

| Discriminant | Type | Rôle |
|---|---|---|
| `0x21` | `RegisterDomain` | réclamation d'un domaine |
| `0x52` | `UpdateDomain` | nouvelle version des données DNS d'un domaine |
| `0x93` | `RegisterTld` | réclamation d'un domaine de premier niveau (M7) |
| `0x47` | `TransferTld` | transfert d'un TLD vers un nouveau propriétaire (M8a) |
| `0x6B` | `RevokeTld` | abandon d'un TLD (M8a) |
| `0xB8` | `SetTldOpen` | ouverture **irréversible** d'un TLD à l'auto-enregistrement (à sens unique, owner-option ; `open=false` est un rejet typé) |
| `0xD4` | `AssignDomain` | attribution directe d'un domaine par le propriétaire du TLD (M8a) |
| `0x3C` | `RenewDomain` | prolongation de l'enregistrement d'un domaine (M8a ; porte le nom canonique depuis owner-option — un TLD ouvert suit l'expiration) |
| `0x8F` | `RenewTld` | renouvellement du terme de 3 ans d'un TLD PRIVÉ (PoW égal à la création) |
| `0x15` | `SlashTx` | preuve d'équivoque d'ancre + bannissement (M9) |
| `0xE2` | `TransferDomain` | transfert d'un domaine vers un nouveau propriétaire (M8c) |
| `0x71` | `Heartbeat` | preuve de vivacité d'un opérateur (RFC proof-of-service) |
| `0xA9` | `SetTldOperator` | délégation de l'exploitation d'un TLD privé (RFC proof-of-service) |
| `0x5E` | `SetDomainOperator` | délégation de l'exploitation d'un domaine (RFC proof-of-service) |

Tout octet de version autre que `0x01` est explicitement rejeté
(`UnsupportedVersion(valeur reçue)`) : un format inconnu ou obsolète
n'est jamais parsé par accident.

## SLASH_TX (0x15, M9)

Preuve on-chain qu'une ancre a signé deux **points de contrôle
conflictuels à la même époque** (équivoque). N'importe qui peut la
soumettre — la preuve est cryptographique et autonome :

```text
offender[32] | evidence_a (CheckpointData 116 B)
  | sig_a[64] | evidence_b (CheckpointData) | sig_b[64]
  | reporter_pk[32] | reporter_signature[64]
```

- `validate()` vérifie la structure : même époque, `signing_hash`
  distincts, deux signatures distinctes, signatures du rapporteur sur
  la charge canonique ;
- règles d'état (journalisées, réversibles sur réorg) : les DEUX
  signatures de l'accusé doivent vérifier contre les DEUX hachages de
  signature distincts à la même époque ; l'accusé doit être dans le
  pool éligible (`SlashOffenderNotInPool` sinon) ; puis
  **bannissement** — la clé est retirée du pool PoS de façon
  persistante (liste noire en état) ;
- sanction minimale : les domaines/TLD de l'accusé ne sont PAS saisis
  (décision M9 documentée) ;
- même point de contrôle deux fois = pas une équivoque (rejeté) ;
  époques différentes = rejeté ; signature forgée = rejeté.

## REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

| Champ | Type | Contraintes |
|---|---|---|
| `name` | `str` ≤ 253 | nom canonique du domaine réclamé, **porté en clair** (voir ci-dessous) |
| `domain_id` | 32 octets bruts | **doit** être égal à `DomainId(name)` (recalculé, jamais cru sur parole) |
| `owner` | 32 octets bruts | **doit** être la dérivation de `public_key` (voir ci-dessous) |
| `timestamp` | varint | information d'ordonnancement (Unix, secondes) |
| `proof` | `bytes` ≤ 256 | charge PoW d'enregistrement (voir [ci-dessous](#preuve-de-travail-pow-denregistrement)) |
| `public_key` | 32 octets bruts | clé publique Ed25519 du signataire (RFC 8032) |
| `signature` | **exactement 64 octets** | signature Ed25519 de la charge signée |

### Le nom est porté par la transaction

`RegisterDomain` transporte le nom canonique en clair, suivi de son
`DomainId`. La cohérence `domain_id == DomainId(name)` est revérifiée
à l'encodage, au décodage et à la validation (défense en profondeur) :
un `RegisterDomain` dont l'id ne correspond pas au nom porté est
invalide. Le `DomainId` reste l'identité de consensus (clés de chaîne
et de DHT) ; le nom porté rend la chaîne auto-descriptive et couvre
toute la réclamation dans la signature.

## REGISTER_TLD (0x93)

Réclamation d'un domaine de premier niveau (registre TLD, M7) :

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) open(0x00|0x01) public_key[32] signature[64]
```

| Champ | Type | Contraintes |
|---|---|---|
| `tld` | `str` ≤ 63 | nom canonique du TLD **porté en clair** (M8b : le défi PoW dérive du nom, comme `RegisterDomain` porte le sien) |
| `tld_id` | 32 octets bruts | identité du TLD : `BLAKE3-256("SCONE-TLD-V1" || tld)` — **doit** être la dérivation de `tld` (recalculée) |
| `owner` | 32 octets bruts | dérivation de `public_key` |
| `timestamp` | varint | information d'ordonnancement (Unix, secondes) |
| `proof` | `bytes` ≤ 256 | charge PoW d'enregistrement (voir [ci-dessous](#preuve-de-travail-pow-denregistrement)) |
| `open` | `0x00\|0x01` | naissance **ouverte** : le TLD atterrit directement dans `TldState::Open` (à personne, libre-service + PoW, un seul PoW au tarif `open_tld_pow_difficulty` — moins cher que privé : ressource commune, ni siège PoS ni exclusivité) ; naissance **privée** : comportement historique (propriétaire, siège PoS, `tld_pow_difficulty`) |
| `public_key` | 32 octets bruts | clé publique Ed25519 du signataire |
| `signature` | **exactement 64 octets** | signature Ed25519 de la charge signée |

L'espace des `TldId` est **disjoint** de celui des `DomainId`
(préfixes de dérivation distincts — voir `/docs/fr/general/naming.md`)
: enregistrer le TLD `uip` (une réclamation sur l'espace de noms
`*.uip`) ne peut jamais squatter l'identité d'un domaine, ni
l'inverse. Règle d'état à l'application : le TLD doit être libre
(registre distinct du registre des domaines).

## La famille M8a — formats wire

Cinq types complètent le registre TLD et le cycle de vie des domaines.
Formats wire normatifs depuis M8a ; **règles d'état complètes depuis
M8b** (voir `/docs/fr/technical/blockchain.md`) : propriétaire de TLD
requis, PoW partout où une réclamation est en libre-service,
expirations et grâce côté domaines. Aucune charge ne porte de champ
`network` sur le wire (il vit uniquement dans la charge de signature).

### TRANSFER_TLD (0x47)

```text
tld_id[32] owner[32] new_owner[32] public_key[32] signature[64]
```

| Champ | Type | Contraintes |
|---|---|---|
| `tld_id` | 32 octets bruts | le TLD transféré |
| `owner` | 32 octets bruts | dérivation de `public_key` (propriétaire actuel / signataire) |
| `new_owner` | 32 octets bruts | identité du destinataire (opaque : toute valeur est syntaxiquement valide ; le lien vers les clés pouvant le dépenser est une règle d'état M8b) |
| `public_key` | 32 octets bruts | clé publique Ed25519 du signataire |
| `signature` | **exactement 64 octets** | signature Ed25519 de la charge signée |

Pas d'horodatage ni de séquence : un TLD a exactement un propriétaire
à la fois ; le premier transfert appliqué gagne (ordre de la chaîne
seul).

### REVOKE_TLD (0x6B)

```text
tld_id[32] owner[32] public_key[32] signature[64]
```

Abandon volontaire d'un TLD (réclamable ensuite par un `RegisterTld`
frais). Signé par le propriétaire actuel ; délibérément sans
horodatage ni séquence — une révocation est ponctuelle contre l'état
courant.

### SET_TLD_OPEN (0xB8)

```text
tld_id[32] owner[32] open u8 open_proof(proof) public_key[32] signature[64]
```

| Champ | Type | Contraintes |
|---|---|---|
| `open` | 1 octet | **strictement `0x00` ou `0x01`** (forme canonique ; toute autre valeur → erreur de décodage, jamais de coercition) |
| `open_proof` | `proof` (varint de longueur + octets) | champ **signé** ; obligatoire et vérifié quand `open = 0x01` (PoW d'ouverture, `open_tld_pow_difficulty` du réseau, défi `SCONE-TLD-OPEN-V1 ‖ tld_id`) ; vide quand `open = 0x00` (la fermeture est gratuite) |

Un TLD **fermé** = attribution uniquement (domaines créés
exclusivement par le propriétaire du TLD via `AssignDomain`) ; un TLD
**ouvert** = n'importe qui peut faire le PoW d'enregistrement et
réclamer un domaine libre (`RegisterDomain`). **L'ouverture** coûte
une preuve de travail supplémentaire (plus légère que la réclamation
du TLD : ~3 jours sur mainnet contre ~1 semaine), payée une fois par
le propriétaire dans `SetTldOpen` — la preuve voyage dans le champ
signé `open_proof` et ne peut pas être substituée après signature.

### ASSIGN_DOMAIN (0xD4)

```text
name(str ≤ 253) domain_id[32] owner[32] assignee[32] public_key[32] signature[64]
```

Miroir de `RegisterDomain` : le nom canonique est porté en clair et
`domain_id` doit être égal à `DomainId(name)` (recalculé au décodage,
à l'encodage et à la validation). Signé par le **propriétaire du TLD
du nom** (vérifié à l'application, M8b) ; `assignee` devient le
premier propriétaire du domaine (opaque, comme `new_owner`).

### RENEW_DOMAIN (0x3C)

```text
name(str ≤ 253) domain_id[32] owner[32] valid_until.v proof(bounded) public_key[32] signature[64]
```

| Champ | Type | Contraintes |
|---|---|---|
| `name` | `str` ≤ 253 | nom canonique **porté en clair** (owner-option : le renouvellement repousse la mort du TLD ouvert sous-jacent, la chaîne doit connaître le nom) ; `domain_id` doit être égal à `DomainId(name)` (recalculé) |
| `valid_until` | varint | nouvelle échéance d'enregistrement (Unix, secondes) |
| `proof` | borné | **autorisation de renouvellement** (voir ci-dessous) |

Signé par le propriétaire actuel du domaine ; savoir si `valid_until`
étend réellement l'échéance (et d'au plus un terme de renouvellement)
est une règle d'état (M8b).

#### Autorisation de renouvellement — PoW ou PoS selon le TLD

Le coût d'un renouvellement dépend de la nature du TLD sous-jacent
(évaluée à l'application, sur l'état du bloc) :

- **TLD ouvert** : le renouvellement coûte le **même PoW que la
  création** — même défi (`SCONE-DOMAIN-V1 ‖ nom canonique`), même
  difficulté par longueur de label et par réseau, même fenêtre de
  validité d'époque. Un domaine n'est jamais « collecté »
  gratuitement : chaque terme de vie est payé en travail.
- **TLD privé** : l'espace de noms est le mise du propriétaire du TLD
  (PoS). La transaction doit être **signée par le propriétaire du
  TLD** (ou son opérateur délégué) — la signature EST l'autorisation,
  le champ `proof` doit être vide (20 octets à zéro). Le propriétaire
  du TLD renouvelle les domaines de son espace sous ses propres
  règles.

### RENEW_TLD (0x8F)

```text
tld(str ≤ 63) tld_id[32] owner[32] valid_until.v proof(bounded) public_key[32] signature[64]
```

| Champ | Type | Contraintes |
|---|---|---|
| `tld` | `str` ≤ 63 | nom canonique du TLD ; `tld_id` doit être égal à `TldId(tld)` (recalculé) |
| `valid_until` | varint | doit être EXACTEMENT `now + 3 years` (terme fixe — redérivé à l'application, toute autre valeur rejetée) |
| `proof` | borné | PoW de renouvellement (voir ci-dessous) |

Signé par le propriétaire **privé** actuel du TLD. Règles
d'application :

- s'applique à un TLD PRIVÉ uniquement (un TLD ouvert vit à travers
  ses domaines — rejet typé `TldIsOpen`) ;
- valable durant le terme ET pendant la **grâce de 60 jours** qui suit
  l'expiration ; au-delà, le TLD est mort (GC) et le rejet est
  `TldExpiredFrozen` ;
- **PoW égal à la création** : défi
  `SCONE-TLD-V1 ‖ nom ‖ expiration_actuelle(BE64)` (le suffixe
  anti-réutilisation lie chaque PoW à UN terme), même difficulté par
  longueur que la réclamation initiale ;
- le renouvellement n'est PAS gelé pendant la grâce — c'est la seule
  opération de gestion encore disponible pour un propriétaire expiré.

## UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

| Champ | Type | Contraintes |
|---|---|---|
| `domain_id` | 32 octets bruts | domaine visé |
| `owner` | 32 octets bruts | dérivation de `public_key` |
| `sequence` | varint | **> 0**, doit être égal à `current + 1` à l'application |
| `record_hash` | 32 octets bruts | engagement vers l'ensemble d'enregistrements DHT |
| `public_key` | 32 octets bruts | clé publique Ed25519 du signataire |
| `signature` | **exactement 64 octets** | signature Ed25519 de la charge signée |

Note : `UpdateDomain` ne porte **aucun** `timestamp` — l'ordonnancement
passe par `sequence`. Il ne porte pas non plus le nom : le domaine est
déjà enregistré, l'id suffit.

### Bornes strictes

- `public_key` : **exactement 32 octets bruts**, pas de préfixe de
  longueur. Une pk qui ne se décode pas comme un point de la courbe →
  erreur `InvalidTransactionKey` au décodage.
- `signature` : **exactement 64 octets bruts**, pas de préfixe de
  longueur, pas de marge. Toute autre taille est une erreur de
  décodage dure (troncature rejetée).
- `proof` : varint de longueur + octets, plafonné à 256
  (`MAX_PROOF_LEN`), vérifié avant allocation.
- `name` : varint de longueur + octets UTF-8, plafonné à 253
  (`MAX_NAME_LEN`), revalidé par les règles de nommage de
  `scone-core`.

## Dérivation du propriétaire

Règle dure : **le propriétaire est toujours recalculé, jamais cru sur
parole**.

```text
PublicKeyRef = BLAKE3-256("SCONE-PUBKEY-V1" || public_key)
OwnerId      = BLAKE3-256("SCONE-OWNER-V1" || PublicKeyRef)
```

Une transaction dont le `owner` ne correspond pas à la dérivation de
sa propre `public_key` est invalide (`InvalidOwner` côté core,
`OwnerKeyMismatch` côté chaîne), rejetée à l'encodage, au décodage
**et** à la validation de chaîne (défense en profondeur).

## Charge signée

La charge signée est l'encodage canonique de la transaction **sans le
champ signature**, préfixée par une étiquette de séparation de
domaine — **avec le `network_id` de la chaîne cible injecté en
tête** :

```text
signing_payload(tx, network) = "SCONE-TX-SIG-V1"
                              || canonical_encode(tx_without_signature, network)
```

où `canonical_encode(tx_without_signature, network)` suit la
structure wire ci-dessus, discriminant et octet de version compris,
en s'arrêtant avant `signature`, **avec `network` comme premier
champ** — c'est le SEUL endroit où le réseau apparaît. La chaîne
recalcule la charge avec SON propre réseau : une transaction signée
pour un autre réseau échoue à la vérification de signature
(`InvalidSignature`) ; c'est le mécanisme anti rejeu inter-chaînes.

```text
REGISTER_DOMAIN_no_sig = disc(0x21) version(0x01) network name domain_id[32] owner[32]
                            timestamp.v proof public_key[32]
REGISTER_TLD_no_sig    = disc(0x93) version(0x01) network tld tld_id[32] owner[32]
                            timestamp.v proof public_key[32]
UPDATE_DOMAIN_no_sig   = disc(0x52) version(0x01) network domain_id[32] owner[32]
                            sequence.v record_hash[32] public_key[32]
TRANSFER_TLD_no_sig    = disc(0x47) version(0x01) network tld_id[32] owner[32]
                            new_owner[32] public_key[32]
REVOKE_TLD_no_sig      = disc(0x6B) version(0x01) network tld_id[32] owner[32]
                            public_key[32]
SET_TLD_OPEN_no_sig    = disc(0xB8) version(0x01) network tld_id[32] owner[32]
                            open(0x00|0x01) open_proof(proof) public_key[32]
ASSIGN_DOMAIN_no_sig   = disc(0xD4) version(0x01) network name domain_id[32] owner[32]
                            assignee[32] public_key[32]
RENEW_DOMAIN_no_sig    = disc(0x3C) version(0x01) network name domain_id[32] owner[32]
                            valid_until.v proof public_key[32]
TRANSFER_DOMAIN_no_sig = disc(0xE2) version(0x01) network domain_id[32] owner[32]
                            new_owner[32] public_key[32]
```

Propriétés :

- déterministe et pur : même transaction ⇒ même charge partout ;
- le préfixe ASCII `SCONE-TX-SIG-V1` (15 octets,
  hex `53434f4e452d54582d5349472d5631`) empêche toute réutilisation de
  la signature dans un autre domaine de hachage du protocole ;
- tous les champs sauf `signature` sont couverts — **y compris le nom
  porté par un `RegisterDomain`** : modifier l'un d'eux quelconque
  invalide la signature ;
- le `network_id` injecté est couvert : la même tx non signée signée
  pour deux réseaux produit deux charges distinctes (donc deux
  signatures distinctes).

## Preuve de travail (PoW) d'enregistrement

Le champ `proof` de `RegisterDomain` et `RegisterTld` porte la preuve
de travail d'enregistrement (module `scone-core::pow`).

### Condensé (sel d'époque, anti pré-minage)

```text
epoch  = block_height / POW_WINDOW        (POW_WINDOW = 144 blocks)
salt   = BLAKE3-256("SCONE-POW-V1-ARGON2" || network || challenge || epoch_le64)
digest = Argon2id(m, t, p = 1; password = nonce_le64, salt = salt[0..16])
```

`network` est le `network_id` canonique (M8b) : une preuve minée pour
un réseau n'est pas une preuve sur un autre.

`SCONE-POW-V1-ARGON2` épingle aussi les paramètres Argon2id par réseau
(`pow_params` : mainnet m = 64 MiB, t = 2, p = 1 ;
testnet/devnet m = 8 KiB, t = 1, p = 1) : le cœur Argon2id (RFC 9106)
rend le coût par hachage dominé par la mémoire (résistance
GPU/ASIC), et une preuve minée sous un jeu de paramètres ne vérifie
jamais sous un autre.

`challenge` est l'entrée de dérivation de l'objet réclamé —
`"SCONE-TLD-V1" || tld` pour un TLD, `"SCONE-DOMAIN-V1" || name` pour
un domaine : exactement les octets hachés pour `TldId` / `DomainId`,
si bien qu'une preuve n'est jamais rejouable entre espaces de noms ni
entre genres d'objets. `epoch_le64` et `nonce_le64` sont les
encodages little-endian sur 8 octets de l'époque et du nonce.

La preuve est valide quand `digest` a au moins `difficulty` bits de
tête à zéro.

#### Sel d'époque — anti pré-minage

Un condensé sans sel ne lie ni le temps ni la hauteur : n'importe qui
pourrait pré-miner aujourd'hui des preuves pour des milliers de noms,
puis les enregistrer en rafale au lancement de la mainnet (squatting)
— la preuve était autonome et valide quel que soit le bloc arrivé en
premier. Le condensé est salé avec l'**époque du bloc qui APPLIQUE la
transaction** :

- `epoch = block_height / POW_WINDOW`, `POW_WINDOW = 144` blocs
  (~2.6 h à 65 s/bloc) : une fenêtre de grâce pour l'utilisateur
  honnête (miner, puis soumettre dans la fenêtre), re-minage
  obligatoire au-delà ;
- la preuve DÉCLARE son époque sur le wire ; à l'application,
  l'époque déclarée doit tomber dans la **fenêtre de validité** du
  bloc appliquant — tolérance vers le PASSÉ uniquement (la
  soumission tardive est absorbée, miner « pour le futur » est
  impossible), fenêtre dimensionnée par
  `epoch_validity_window(difficulty)` = `1 << max(0, difficulty −
  16)` époques (fenêtre élargie : les hautes difficultés mainnet se
  minent en jours à semaines, plus longtemps qu'une époque ; le
  stockage ne gagne rien car la fenêtre ≈ le coût de minage — chaque
  preuve d'une rafale coûte ses `2^difficulty` hachages complets DANS
  la fenêtre). À difficulté ≤ 16 bits (testnet, noms longs), la règle
  d'égalité stricte est inchangée ;
- admission au mempool (au mieux) : la hauteur du bloc futur est
  inconnue — le relais vérifie contre l'époque de `hauteur courante +
  1` ; la vérification stricte au moment de l'application reste
  l'autorité ;
- rejeu déterministe : la preuve PORTE son époque et la fenêtre est
  une fonction pure de (difficulté, époque du bloc) — un bloc
  historique se revérifie à l'identique sur tout nœud.

Changement cassant pré-mainnet : l'ancien format compact (12 octets)
est rejeté sans rétrocompatibilité.

### Charge wire (20 octets fixes)

```text
nonce_le64 || epoch_le64 || difficulty_le32
```

La difficulté et l'époque voyagent avec la preuve, mais elles sont
**déclarées par le producteur donc jamais crues sur parole** : le
vérificateur compare la difficulté à la **constante de protocole**
pour le genre d'enregistrement (une charge dont la difficulté
déclarée diffère de la constante — plus basse *ou* plus haute — est
rejetée, `InvalidProof`), et l'époque à celle du bloc appliquant
(`ProofEpochMismatch` en cas d'écart). Le nonce est ensuite
revérifié à la difficulté constante, condensé recalculé from scratch.

### Difficultés (courbe **fondée sur la longueur**, M8b + calibration mesurée)

La difficulté n'est PAS un paramètre plat : c'est une fonction
déterministe de la **longueur du label créé** (`scone-core::pow_length`
— le TLD pour `RegisterTld`, le label le plus à gauche pour
`RegisterDomain`), par réseau :

```text
difficulty(len) = apex − round(0.415 × (len − 1)), clamped at the floor
```

| Réseau | TLD privé (apex/plancher) | TLD né ouvert / domaine (apex/plancher) |
|---|---|---|
| testnet | 8 bits / 2 bits | 6 bits / 2 bits |
| mainnet | 42 bits / 30 bits | 40 bits / 30 bits |
| devnet | 2 bits / 1 bit | 1 bit / 1 bit |

Un TLD **né ouvert** (`RegisterTld` avec `open = 0x01`) paie la courbe
« ouverture » (apex − 2, comme `domain_difficulty`) ; **ouvrir un TLD
privé existant** (`SetTldOpen`) reste à difficulté plate
(`open_tld_pow_difficulty` : 6 bits testnet / 22 bits mainnet /
2 bits devnet — la tx ne porte que le `tld_id`, le nom a payé sa
longueur à la création). Benchmarks mainnet (TLD privé, ~6 M
hachages/s) : 1 caractère = 42 bits (≥ 7 jours), 5 caractères = 40
bits (~2 jours), 19 caractères = 35 bits (~1.6 h), 29+ caractères =
plancher 30 bits (~3 min). Les courbes vivent dans
`scone-core::pow_length` (constantes `MAINNET_APEX = 42`,
`MAINNET_FLOOR = 30`, `TESTNET_APEX = 8`, `TESTNET_FLOOR = 2`,
`DEVNET_APEX = 2`, `DEVNET_FLOOR = 1`) ; les paramètres plats
(`tld_pow_difficulty`, etc.) restent dans
`scone-core::network::NetworkParams`. Modifier une valeur = un
changement de consensus **pour ce réseau**. Invariant : la devnet est
une troisième `NetworkParams` complète et la testnet reste
strictement identique à ses constantes pré-devnet (paramètres
Argon2id, courbes, fenêtres) — la devnet ne partage AUCUNE constante
avec les autres réseaux au-delà du code lui-même.

Calibration mesurée des sommets plats (2026-10, `cargo run
--release -p scone-core --example pow_bench` — débit mono-thread
~7.7 M H/s sur un x86-64 de référence) :

| Genre (mainnet) | difficulté | temps réel moyen |
|---|---|---|
| `RegisterTld` | 24 bits | ~7 s |
| `RegisterDomain` | 20 bits | ~54 ms |

À ~65 s/bloc : une réclamation de TLD coûte ~0.1 bloc de CPU
mono-cœur, un domaine ~1 ms de bloc — symbolique pour un utilisateur,
coûteux à industrialiser à grande échelle (anti-squat). À recalibrer
si les cibles matérielles baissent (ARM mobile : ~4–8× plus lent).

Branches d'état (M8b, livré) : `RegisterTld` exige TOUJOURS son PoW ;
`RegisterDomain` l'exige sur un TLD **ouvert** (le chemin fermé est à
attribution uniquement — `AssignDomain`, sans PoW, signé par le
propriétaire du TLD).

## Règles de validation

Une transaction est valide si et seulement si :

1. **décodage strict** : structure et bornes ci-dessus respectées
   (discriminant connu, version `0x01`, pk 32 o, signature 64 o,
   invariants core revérifiés au décodage) ;
2. **lien propriétaire/pk** : `owner == OwnerId::from(public_key)`
   (recalculé) ;
3. **cohérence nom/id** (RegisterDomain) : `domain_id ==
   DomainId(name)` (recalculé) ;
4. **signature** : Ed25519 `verify_strict` de `signature` contre la
   charge signée **recalculée from scratch** avec le réseau de la
   chaîne (jamais prise d'un pair ni de la transaction telle que
   fournie) — une tx signée pour un autre réseau échoue ici
   (`InvalidSignature`, anti rejeu inter-chaînes) ;
5. **règles d'état** (à l'application dans un bloc — M8b, voir
   `/docs/fr/technical/blockchain.md`) :
   RegisterTld → TLD libre + non ICANN + PoW (difficulté fondée sur
   la longueur) ; RegisterDomain → TLD enregistré **et ouvert** +
   domaine libre (grâce comprise) + PoW (fondée sur la longueur) ;
   UpdateDomain → domaine existant, `owner` actuel,
   `sequence == current + 1` ;
   TransferTld / RevokeTld / SetTldOpen / AssignDomain → TLD existant
   **et privé**, signataire = le propriétaire du TLD ;
   TransferDomain → domaine existant, propriétaire ;
   RenewDomain → domaine existant, extension stricte ≤ 3 ans,
   autorisation PoW (TLD ouvert : même PoW que la création, **liée à
   l'échéance actuelle** — un défi distinct de l'enregistrement et de
   chaque renouvellement antérieur) ou PoS (TLD privé : signature du
   propriétaire du TLD — ou de son opérateur effectif —, `proof`
   vide).

La vérification utilise `verify_strict` (rejet des signatures
malléables et des clés faibles/non canoniques), comme requis pour des
données non fiables.

## Ordre des opérations dans `push_block`

La validation cryptographique (2 à 4) a lieu pour chaque transaction
**avant** l'application d'état, sur un état de travail atomique par
bloc ; toute erreur laisse la chaîne rigoureusement inchangée.

## Assemblage des blocs

`BlockBuilder` (`scone-blockchain`) assemble un bloc à partir d'une
file de transactions : `tx_root` recalculé sur la liste ordonnée,
en-tête chaîné sur le hachage parent (`prev_hash`), `version` =
`PROTOCOL_VERSION`. La charge de consensus (charge de producteur
signé) et l'horodatage sont fournis par l'appelant ; le trait
`Consensus` existant est utilisé symétriquement avec la validation.

## Exemples hex (vecteurs réels, reproductibles — M8b)

### RegisterDomain `example.uip`, horodatage 42, testnet

Charge signée (vecteur épinglé ; historiquement générée par
l'outillage hors-ligne retiré — les valeurs restent des vecteurs
valides). Le `network` apparaît ici car c'est la CHARGE DE SIGNATURE
(son unique place) ; la transaction wire signée elle-même n'en
contient pas :

```text
53434f4e452d54582d5349472d5631       SCONE-TX-SIG-V1 prefix
21                                    disc REGISTER_DOMAIN (0x21)
01                                    format version
0d 73636f6e652d746573746e6574         network (str, "scone-testnet")
0b 6578616d706c652e756970             name (str, length 11)
e6a2cbe264f81ed7da4d901c4deec6a3579224b8080fad01cb7461b8c264f181  domain_id
7b2446dadddf640244ac8677a401f148b2b23d1c836d5ad3dd340f8ef2edbed8   owner (fixed build key)
2a                                    timestamp (varint, 42)
14 0200000000000000 0000000000000000 04000000
                                      proof (20 B: nonce 2, epoch 0, difficulty 4 — testnet)
2a002152f8d19b791d24453242e15f2eab6cb7cffa7b6a5ed30097960e069881db12  pk + start of sig
```

### RegisterTld `uip`, horodatage 42

Charge signée (vecteur épinglé) — le `TldId` est le vecteur de
dérivation épinglé dans `/docs/fr/general/naming.md` :

```text
53434f4e452d54582d5349472d5631       SCONE-TX-SIG-V1 prefix
93 01                                 disc REGISTER_TLD + version
0d 73636f6e652d746573746e6574         network (str, "scone-testnet")
03 756970                             tld (str, "uip")
ad5a86d68643d5c22d6a959bb1a315530c77dfda241f1ac1a8107450f3fab25e  tld_id ("uip")
7b2446dadddf640244ac8677a401f148b2b23d1c836d5ad3dd340f8ef2edbed8   owner
2a                                    timestamp (varint, 42)
14 8e00000000000000 0000000000000000 07000000
                                      proof (20 B: nonce 0x8e, epoch 0, difficulty 7 — testnet, 3-char label)
00                                    open (0x00 = private birth)
2a002152f8d19b791d24453242e15f2eab6cb7cffa7b6a5ed30097960e069881db12  pk
```

(Les valeurs complètes étaient reproductibles via l'outillage
hors-ligne retiré ; les vecteurs ci-dessous restent épinglés et
valides. Le PoW est miné automatiquement à la difficulté du réseau —
les nonces varient d'un run à l'autre ; seuls réseau/nom/id/owner/pk
sont stables. Depuis le changement wire 2026-09, l'hex de la
transaction signée ne contient PLUS les octets
`0d 73636f6e652d746573746e6574` : le réseau voyage uniquement dans la
charge à signer.)

## Où vit l'hex désormais

Les commandes hors-ligne `tx build|sign|verify` ont été retirées
(cli-simplify-v1). Le format hex signé lui-même est inchangé et reste
le format wire du **flux web/API** : l'application web registrar
construit, signe (coffre-fort dans le navigateur) et soumet les
transactions en hex canonique via l'API. Côté CLI, les parcours
utilisateur sont les formes en une commande :

```bash
scone domain register example.uip        # mines, signs, submits
scone renew example.uip                  # renewal proof, signs, submits
scone domains                            # what you own
```

Dans tous les chemins de code, le propriétaire et la clé sont
**recalculés** depuis l'identité de signature, jamais pris d'une
quelconque entrée.

## Historique des versions

| Version | Contenus | Statut |
|---|---|---|
| 1 | `public_key` + `signature` embarqués, charge `SCONE-TX-SIG-V1`, octet de version `0x01`, `RegisterDomain` porte le nom, `RegisterTld` (0x93), famille M8a : `TransferTld` (0x47), `RevokeTld` (0x6B), `SetTldOpen` (0xB8), `AssignDomain` (0xD4), `RenewDomain` (0x3C) — formats wire normatifs ; M8b : champ `network` signé en tête de toute charge, `RegisterTld` porte le nom, règles d'état complètes, difficultés PoW par réseau | remplacée |
| 1 (2026-09, wire sans réseau) | **retrait du champ `network` du wire de toutes les transactions** : le network_id vit uniquement dans la charge de signature (`signing_payload(tx, network)` l'injecte en tête) ; la séparation des réseaux repose sur la signature (`InvalidSignature` sur une charge étrangère, l'erreur `WrongNetwork` disparaît) ; `TX_FORMAT_VERSION` reste `0x01` (testnet relançable, pas de coexistence) ; **tous les TxId changent** (le TxId est le hachage de l'encodage wire signé, qui perd 14–15 octets) ; condensé PoW, genèse et répertoires de données par réseau inchangés | remplacée |
| 1 (2026-10, sel d'époque PoW) | **condensé PoW anti pré-minage** : sel `BLAKE3("SCONE-POW-V1-ARGON2" ‖ network ‖ challenge ‖ epoch_le)` + cœur Argon2id sur `nonce_le`, avec `epoch = block_height / 144` du bloc qui APPLIQUE la tx ; charge wire de preuve 12 → **20 octets** (`nonce_le ‖ epoch_le ‖ difficulty_le`), l'époque déclarée est vérifiée contre la fenêtre de validité d'époque du bloc appliquant (rejet typé `ProofEpochMismatch`), au mieux `(height+1)/144` à l'admission mempool, drapeau CLI `--epoch` sur les chemins hors-ligne ; changement cassant pré-mainnet : l'ancienne charge de 12 octets est rejetée, **tous les TxId changent** (le champ `proof` grandit de 8 octets) | actuelle |

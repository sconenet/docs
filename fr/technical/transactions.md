# Transactions signées (format v1)

Document de référence **normatif** du format de transaction signée.
Il spécifie les champs, l'ordre wire exact, les tailles, le payload
signé et les règles de validation, de manière à permettre une
implémentation indépendante sans lire le Rust. L'implémentation de
référence est [`scone-protocol`](../../../crates/scone-protocol)
(encodage) et [`scone-blockchain`](../../../crates/scone-blockchain)
(validation).

`PROTOCOL_VERSION` = **1**. Il n'existe qu'un seul ensemble de
formats : tout octet de version autre que `0x01` (dans une
transaction) ou différent de la version locale (dans un bloc) est
rejeté avec `UnsupportedVersion`.

## Vue d'ensemble

```text
Transaction = disc u8 || version u8 || payload
```

| Champ | Taille | Rôle |
|---|---|---|
| `disc` | 1 o | type de transaction (table ci-dessous) |
| `version` | 1 o | **exactement `0x01`** (version du format tx) |

**Les transactions ne portent plus de champ `network` sur le wire**
(changement 2026-09) : le `network_id` (`str` ≤ 16, ASCII
`[a-z0-9-]` — ex. `scone-testnet`, 13 octets) vit **uniquement dans
le payload de signature** — il y est injecté comme premier champ par
le signataire (voir [Payload signé](#payload-signé)) — et reste séparé
par réseau dans le digest PoW. Une transaction **signée** pour un
réseau ne vérifie donc jamais sur un autre : sa signature échoue
(`InvalidSignature`) sur le payload recomputé par la chaîne, qui
injecte son propre réseau (voir `/docs/fr/technical/blockchain.md` —
réseaux et genèse).

### Discriminants (table normative)

Chaque discriminant est une **constante opaque, fixe et arbitraire** :
les valeurs sont choisies distinctes exprès, pour que le wire ne
suggère **jamais** d'ordre logique entre les types (aucune suite
0x01/0x02/0x03…). `0x00` n'est **jamais** un type valide (octet de
corruption typique). Les types futurs prendront toute valeur libre
non utilisée, documentée une fois pour toutes dans cette table.

| Discriminant | Type | Rôle |
|---|---|---|
| `0x21` | `RegisterDomain` | claim d'un domaine |
| `0x52` | `UpdateDomain` | nouvelle version des données DNS d'un domaine |
| `0x93` | `RegisterTld` | claim d'un top-level domain (M7) |
| `0x47` | `TransferTld` | transfert d'un TLD à un nouvel owner (M8a) |
| `0x6B` | `RevokeTld` | abandon d'un TLD (M8a) |
| `0xB8` | `SetTldOpen` | ouverture **irréversible** d'un TLD à l'auto-enregistrement (one-way, owner-option ; `open=false` rejeté typé) |
| `0xD4` | `AssignDomain` | assignation directe d'un domaine par l'owner du TLD (M8a) |
| `0x3C` | `RenewDomain` | prolongation de l'enregistrement d'un domaine (M8a ; porte le nom canonique depuis owner-option — le TLD ouvert suit l'échéance) |
| `0x15` | `SlashTx` | preuve d'équivocation d'un anchor + bannissement (M9) |
| `0xE2` | `TransferDomain` | transfert d'un domaine à un nouvel owner (M8c) |
| `0x71` | `Heartbeat` | preuve de vie d'un opérateur (RFC proof-of-service) |
| `0xA9` | `SetTldOperator` | délégation de l'exploitation d'un TLD privé (RFC proof-of-service) |
| `0x5E` | `SetDomainOperator` | délégation de l'exploitation d'un domaine (RFC proof-of-service) |

Tout octet de version autre que `0x01` est rejeté explicitement
(`UnsupportedVersion(valeur reçue)`) : un format inconnu ou obsolète
n'est jamais analysé par accident.

## SLASH_TX (0x15, M9)

Preuve on-chain qu'un anchor a signé deux checkpoints
**conflictuels à la même epoch** (équivocation). Chacun peut la
soumettre — la preuve est cryptographique et autonome :

```text
offender[32] | evidence_a (CheckpointData 116 o)
  | sig_a[64] | evidence_b (CheckpointData) | sig_b[64]
  | reporter_pk[32] | reporter_signature[64]
```

- `validate()` vérifie la structure : même epoch, `signing_hash`
  distincts, deux signatures distinctes, signatures du reporter sur
  le payload canonique ;
- règles d'état (journalisées, annulables au reorg) : les DEUX
  signatures de l'accusé doivent vérifier sur les DEUX signing hashes
  distincts à la même epoch ; l'accusé doit être dans le pool
  éligible (`SlashOffenderNotInPool` sinon) ; puis **bannissement**
  — la clé est retirée du pool PoS de façon persistante (liste noire
  dans l'état) ;
- sanction minimale : les domaines/TLD de l'accusé ne sont PAS saisis
  (décision M9 documentée) ;
- même checkpoint deux fois = pas une équivocation (rejet) ; epochs
  différentes = rejet ; signature forgée = rejet.

## REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

| Champ | Type | Contraintes |
|---|---|---|
| `name` | `str` ≤ 253 | nom canonique du domaine claimé, **porté en clair** (voir ci-dessous) |
| `domain_id` | 32 octets nus | **doit** valoir `DomainId(name)` (recomputé, jamais cru) |
| `owner` | 32 octets nus | **doit** être la dérivation de `public_key` (voir ci-dessous) |
| `timestamp` | varint | information d'ordre (Unix, secondes) |
| `proof` | `bytes` ≤ 256 | charge utile du PoW de registration (voir [ci-dessous](#preuve-de-travail-pow-de-registration)) |
| `public_key` | 32 octets nus | clé publique Ed25519 du signataire (RFC 8032) |
| `signature` | **exactement 64 octets** | signature Ed25519 du payload signé |

### Le nom est porté par la transaction

Le `RegisterDomain` transporte le nom canonique en clair, suivi de
son `DomainId`. La cohérence `domain_id == DomainId(name)` est
re-vérifiée à l'encodage, au décodage et à la validation (défense en
profondeur) : un `RegisterDomain` dont l'id ne correspond pas au nom
porté est invalide. Le `DomainId` reste l'identité de consensus
(clés de chaîne, de DHT) ; le nom porté rend la chaîne
auto-descriptive et couvre l'intégralité de la claim dans la
signature.

## REGISTER_TLD (0x93)

Claim d'un top-level domain (registre TLD, M7) :

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) open(0x00|0x01) public_key[32] signature[64]
```

| Champ | Type | Contraintes |
|---|---|---|
| `tld` | `str` ≤ 63 | nom canonique du TLD **porté en clair** (M8b : le challenge PoW se dérive du nom, comme `RegisterDomain` porte le sien) |
| `tld_id` | 32 octets nus | identité du TLD : `BLAKE3-256("SCONE-TLD-V1" || tld)` — **doit** être la dérivation de `tld` (recomputé) |
| `owner` | 32 octets nus | dérivation de `public_key` |
| `timestamp` | varint | information d'ordre (Unix, secondes) |
| `proof` | `bytes` ≤ 256 | charge utile du PoW de registration (voir [ci-dessous](#preuve-de-travail-pow-de-registration)) |
| `open` | `0x00\|0x01` | naissance **ouverte** : le TLD atterrit directement en `TldState::Open` (personne, self-service + PoW, un seul PoW au tarif `open_tld_pow_difficulty` — moins cher que le privé : ressource commune, ni siège PoS ni exclusivité) ; naissance **privée** : comportement historique (propriétaire, siège PoS, `tld_pow_difficulty`) |
| `public_key` | 32 octets nus | clé Ed25519 du signataire |
| `signature` | **exactement 64 octets** | signature Ed25519 du payload signé |

L'espace des `TldId` est **disjoint** de celui des `DomainId`
(préfixes de dérivation distincts — voir `/docs/fr/general/naming.md`) :
enregistrer le TLD `uip` (claim sur le namespace `*.uip`) ne peut
jamais squatter l'identité d'un domaine, ni l'inverse. Règle d'état à
l'application : le TLD doit être libre (registre séparé de celui des
domaines).

## Famille M8a — formats wire

Cinq types complètent le registre TLD et le cycle de vie des
domaines. Formats wire normatifs depuis M8a ; **règles d'état
complètes depuis M8b** (voir `/docs/fr/technical/blockchain.md`) :
owner-du-TLD requis, PoW là où une claim est auto-service,
expirations et grâce côté domaines. Aucun payload ne porte de champ
`network` sur le wire (il vit dans le seul payload de signature).

### TRANSFER_TLD (0x47)

```text
tld_id[32] owner[32] new_owner[32] public_key[32] signature[64]
```

| Champ | Type | Contraintes |
|---|---|---|
| `tld_id` | 32 octets nus | TLD transféré |
| `owner` | 32 octets nus | dérivation de `public_key` (owner courant / signataire) |
| `new_owner` | 32 octets nus | identité du destinataire (opaque : toute valeur est syntaxiquement valide ; le binding aux clés qui peuvent le dépenser est une règle d'état M8b) |
| `public_key` | 32 octets nus | clé Ed25519 du signataire |
| `signature` | **exactement 64 octets** | signature Ed25519 du payload signé |

Pas de timestamp ni de séquence : un TLD a exactement un owner à la
fois, le premier transfert appliqué gagne (ordre de chaîne seul).

### REVOKE_TLD (0x6B)

```text
tld_id[32] owner[32] public_key[32] signature[64]
```

Abandon volontaire d'un TLD (re-claimable ensuite par un
`RegisterTld` frais). Signé par l'owner courant ; volontairement sans
timestamp ni séquence — un revoke est one-shot contre l'état courant.

### SET_TLD_OPEN (0xB8)

```text
tld_id[32] owner[32] open u8 open_proof(proof) public_key[32] signature[64]
```

| Champ | Type | Contraintes |
|---|---|---|
| `open` | 1 octet | **strictement `0x00` ou `0x01`** (forme canonique ; toute autre valeur → erreur de décodage, jamais de coercion) |
| `open_proof` | `proof` (varint-longueur + octets) | champ **signé** ; obligatoire et vérifié quand `open = 0x01` (PoW d'ouverture, difficulté `open_tld_pow_difficulty` du réseau, challenge `SCONE-TLD-OPEN-V1 ‖ tld_id`) ; vide quand `open = 0x00` (fermer est gratuit) |

TLD **fermé** = assign-only (domaines créés exclusivement par l'owner
du TLD via `AssignDomain`) ; TLD **ouvert** = quiconque peut faire le
PoW de registration et claimer un domaine libre (`RegisterDomain`).
**Ouvrir** coûte une preuve de travail supplémentaire (plus légère que
la claim du TLD : ~3 jours sur mainnet contre ~1 semaine), payée une
fois par l'owner dans `SetTldOpen` — la preuve voyage dans le champ
signé `open_proof` et ne peut pas être substituée après signature.

### ASSIGN_DOMAIN (0xD4)

```text
name(str ≤ 253) domain_id[32] owner[32] assignee[32] public_key[32] signature[64]
```

Miroir de `RegisterDomain` : le nom canonique est porté en clair et
`domain_id` doit valoir `DomainId(name)` (recomputé au décodage,
encodage et validation). Signé par l'**owner du TLD** du nom
(vérifié à l'application, M8b) ; `assignee` devient le premier owner
du domaine (opaque, comme `new_owner`).

### RENEW_DOMAIN (0x3C)

```text
name(str ≤ 253) domain_id[32] owner[32] valid_until.v proof(borné) public_key[32] signature[64]
```

| Champ | Type | Contraintes |
|---|---|---|
| `name` | `str` ≤ 253 | nom canonique **porté en clair** (owner-option : le renouvellement reporte la mort du TLD ouvert sous-jacent, la chaîne doit connaître le nom) ; `domain_id` doit valoir `DomainId(name)` (recomputé) |
| `valid_until` | varint | nouvelle échéance d'enregistrement (Unix, secondes) |
| `proof` | borné | **autorisation de renouvellement** (voir ci-dessous) |

Signé par l'owner courant du domaine ; le fait que `valid_until`
prolonge réellement l'échéance (et d'au plus un terme de
renouvellement) est une règle d'état (M8b).

#### Autorisation de renouvellement — PoW ou PoS selon le TLD

Le coût d'un renouvellement dépend de la nature du TLD sous-jacent
(évaluée à l'application, sur l'état du bloc) :

- **TLD ouvert** : le renouvellement coûte la **même PoW que la
  création** — même challenge (`SCONE-DOMAIN-V1 ‖ nom canonique`),
  même difficulté par longueur de label et par réseau, même fenêtre
  de validité d'époque. Un domaine n'est jamais « collectionné »
  gratuitement : chaque terme de vie est payé en travail.
- **TLD privé** : le namespace est le stake du owner du TLD (PoS).
  La transaction doit être **signée par le owner du TLD** (ou son
  opérateur délégué) — la signature EST l'autorisation, le champ
  `proof` doit être vide (20 octets nuls). Le owner du TLD renouvelle
  les domaines de son espace selon ses propres règles.

## UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

| Champ | Type | Contraintes |
|---|---|---|
| `domain_id` | 32 octets nus | domaine visé |
| `owner` | 32 octets nus | dérivation de `public_key` |
| `sequence` | varint | **> 0**, doit valoir `current + 1` à l'application |
| `record_hash` | 32 octets nus | engagement vers le record set DHT |
| `public_key` | 32 octets nus | clé Ed25519 du signataire |
| `signature` | **exactement 64 octets** | signature Ed25519 du payload signé |

Note : `UpdateDomain` ne porte **pas** de `timestamp` — l'ordre passe
par `sequence`. Il ne porte pas non plus le nom : le domaine est déjà
enregistré, l'id suffit.

### Bornes strictes

- `public_key` : **exactement 32 octets bruts**, sans préfixe de
  longueur. Une pk qui ne se décode pas en point de courbe → erreur
  `InvalidTransactionKey` au décodage.
- `signature` : **exactement 64 octets bruts**, sans préfixe de
  longueur ni marge. Toute autre taille est une erreur dure de
  décodage (troncature rejetée).
- `proof` : varint longueur + octets, plafonné à 256
  (`MAX_PROOF_LEN`), vérifié avant allocation.
- `name` : varint longueur + octets UTF-8, plafonné à 253
  (`MAX_NAME_LEN`), revalidé par les règles de nommage `scone-core`.

## Dérivation de l'owner

Règle dure : **l'owner est toujours recalculé, jamais cru**.

```text
PublicKeyRef = BLAKE3-256("SCONE-PUBKEY-V1" || public_key)
OwnerId      = BLAKE3-256("SCONE-OWNER-V1" || PublicKeyRef)
```

Une transaction dont `owner` ne correspond pas à la dérivation de sa
propre `public_key` est invalide (`InvalidOwner` côté core,
`OwnerKeyMismatch` côté chaîne), rejetée à l'encodage, au décodage
**et** à la validation de chaîne (défense en profondeur).

## Payload signé

Le payload signé est l'encodage canonique de la transaction **sans le
champ signature**, préfixé par un domaine de séparation — **avec le
`network_id` de la chaîne ciblée injecté en tête** :

```text
signing_payload(tx, network) = "SCONE-TX-SIG-V1"
                              || canonical_encode(tx_sans_signature, network)
```

où `canonical_encode(tx_sans_signature, network)` reprend la
structure wire ci-dessus, discriminant et octet de version compris,
en s'arrêtant avant `signature`, **avec `network` comme premier
champ** — c'est la SEULE place où le network apparaît. La chaîne
recompute le payload avec SON propre réseau : une transaction signée
pour un autre réseau échoue à la vérification de signature
(`InvalidSignature`) ; c'est le mécanisme anti-replay cross-chain.

```text
REGISTER_DOMAIN_sans_sig = disc(0x21) version(0x01) network name domain_id[32] owner[32]
                            timestamp.v proof public_key[32]
REGISTER_TLD_sans_sig    = disc(0x93) version(0x01) network tld tld_id[32] owner[32]
                            timestamp.v proof public_key[32]
UPDATE_DOMAIN_sans_sig   = disc(0x52) version(0x01) network domain_id[32] owner[32]
                            sequence.v record_hash[32] public_key[32]
TRANSFER_TLD_sans_sig    = disc(0x47) version(0x01) network tld_id[32] owner[32]
                            new_owner[32] public_key[32]
REVOKE_TLD_sans_sig      = disc(0x6B) version(0x01) network tld_id[32] owner[32]
                            public_key[32]
SET_TLD_OPEN_sans_sig    = disc(0xB8) version(0x01) network tld_id[32] owner[32]
                            open(0x00|0x01) open_proof(proof) public_key[32]
ASSIGN_DOMAIN_sans_sig   = disc(0xD4) version(0x01) network name domain_id[32] owner[32]
                            assignee[32] public_key[32]
RENEW_DOMAIN_sans_sig    = disc(0x3C) version(0x01) network name domain_id[32] owner[32]
                            valid_until.v proof public_key[32]
TRANSFER_DOMAIN_sans_sig = disc(0xE2) version(0x01) network domain_id[32] owner[32]
                            new_owner[32] public_key[32]
```

Propriétés :

- déterministe et pur : même transaction ⇒ même payload partout ;
- le préfixe ASCII `SCONE-TX-SIG-V1` (15 octets,
  hex `53434f4e452d54582d5349472d5631`) empêche toute réutilisation
  de la signature dans un autre domaine de hash du protocole ;
- tous les champs sauf `signature` sont couverts — **y compris le
  nom porté par un `RegisterDomain`** : modifier n'importe lequel
  invalide la signature ;
- le `network_id` injecté est couvert : le même unsigned tx signé
  pour deux réseaux produit deux payloads (donc deux signatures)
  distincts.

## Preuve de travail (PoW) de registration

Le champ `proof` de `RegisterDomain` et `RegisterTld` porte la preuve
de travail de registration (module `scone-core::pow`).

### Digest (V2 — sel d'époque, anti pré-minage)

```text
epoch  = block_height / POW_WINDOW        (POW_WINDOW = 144 blocs)
digest = BLAKE3-256("SCONE-POW-V2" || network || challenge || epoch_le64 || nonce_le64)
```

`network` est le `network_id` canonique (M8b) : une preuve minée pour
un réseau n'est pas une preuve sur un autre.

`challenge` est l'entrée de dérivation de l'objet claimé —
`"SCONE-TLD-V1" || tld` pour un TLD, `"SCONE-DOMAIN-V1" || nom` pour
un domaine : exactement les octets hashés pour `TldId` / `DomainId`,
si bien qu'une preuve n'est jamais rejouable entre namespaces ni
entre kinds d'objets. `epoch_le64` et `nonce_le64` sont les encodages
little-endian sur 8 octets de l'époque et du nonce.

La preuve est valide quand `digest` a au moins `difficulty` bits de
zéro en tête.

#### Sel d'époque — anti pré-minage / Epoch salt — anti pre-mining

Le digest V1 ne liait ni le temps ni la hauteur : n'importe qui
pouvait pré-miner aujourd'hui des preuves pour des milliers de noms
puis les enregistrer en rafale au lancement mainnet (squattage) — la
preuve était autonome et valait au premier bloc venu. Le digest V2
sale la preuve avec l'**époque du bloc qui APPLIQUE la transaction**
:

- `epoch = block_height / POW_WINDOW`, `POW_WINDOW = 144` blocs
  (~2,6 h à 65 s/bloc) : une fenêtre de grâce pour l'utilisateur
  honnête (miner, puis soumettre dans la fenêtre), un re-minage
  obligatoire au-delà / a grace window for the honest user, mandatory
  re-mining beyond it ;
- la preuve DÉCLARE son époque sur le wire ; à l'application,
  l'époque déclarée doit tomber dans la **fenêtre de validité** du
  bloc appliquant — tolérance **arrière uniquement** (on peut
  soumettre en retard, jamais miner « pour le futur »), fenêtre
  dimensionnée par `epoch_validity_window(difficulty)` = `1 <<
  max(0, difficulty − 16)` époques (élargissement V3 : les hautes
  difficultés mainnet se minent en jours—semaines, plus longtemps
  qu'une époque ; le stock de preuves n'y gagne rien car la fenêtre
  ≈ le coût de minage — chaque preuve d'une rafale coûte
  intégralement ses `2^difficulty` haches DANS la fenêtre) / the
  declared epoch must fall in the applying block's **validity
  window** — BACKWARD tolerance only (late submission is absorbed,
  mining "for the future" is impossible), window sized by
  `epoch_validity_window(difficulty)` = `1 << max(0, difficulty −
  16)` epochs (V3 widening: high mainnet difficulties mine in
  days-to-weeks, longer than one epoch; stockpiling gains nothing
  because the window ≈ the mining cost). À difficulté ≤ 16 bits
  (testnet, noms longs) la règle V2 d'égalité stricte est
  inchangée / at difficulty ≤ 16 bits (testnet, long names) the V2
  strict-equality rule is unchanged ;
- admission mempool (best effort) : la hauteur future du bloc est
  inconnue — le relais vérifie contre l'époque de `hauteur courante +
  1`, la vérification stricte à l'application reste l'autorité /
  mempool admission (best effort): the relay verifies against
  `epoch(current_height + 1)`; strict application-time verification
  remains the authority ;
- rejeu déterministe : la preuve PORTE son époque, la fenêtre est
  une fonction pure de (difficulté, époque du bloc) — un bloc
  historique re-vérifie à l'identique sur tout nœud.

Changement cassant pré-mainnet : le format V1 (12 octets) est rejeté
sans rétrocompatibilité.

### Charge utile wire (20 octets fixes)

```text
nonce_le64 || epoch_le64 || difficulty_le32
```

La difficulté et l'époque voyagent avec la preuve, mais elles sont
**producteur-déclarées et donc jamais crues** : le vérificateur
compare la difficulté à la **constante protocole** du kind de
registration (un payload dont la difficulté déclarée diffère de la
constante — plus basse *ou* plus haute — est rejeté,
`InvalidProof`), et l'époque à celle du bloc appliquant
(`ProofEpochMismatch` en cas d'écart). Le nonce est ensuite
re-vérifié à la difficulté constante, digest recomputé de zéro.

### Difficultés (courbe **par longueur**, M8b + calibrage mesuré)

La difficulté n'est PAS un paramètre plat : c'est une fonction
déterministe de la **longueur du label créé**
(`scone-core::pow_length` — le TLD pour `RegisterTld`, le label de
gauche pour `RegisterDomain`), par réseau :

```text
difficulté(len) = apex − round(0,415 × (len − 1)), bornée au plancher
```

| Réseau | TLD privé (apex/plancher) | TLD né ouvert / domaine (apex/plancher) |
|---|---|---|
| testnet | 8 bits / 2 bits | 6 bits / 2 bits |
| mainnet | 42 bits / 30 bits | 40 bits / 30 bits |
| devnet | 2 bits / 1 bit | 1 bit / 1 bit |

Un TLD **né ouvert** (`RegisterTld` avec `open = 0x01`) paie la courbe
« ouverture » (apex − 2, comme `domain_difficulty`) ; **ouvrir un TLD
privé existant** (`SetTldOpen`) reste à difficulté plate
(`open_tld_pow_difficulty` : 6 bits testnet / 22 bits mainnet / 2 bits
devnet — la tx
ne porte que le `tld_id`, le nom a payé sa longueur à la création).
Repères mainnet (TLD privé, ~6 M hashs/s) : 1 caractère = 42 bits
(≥ 7 jours), 5 car = 40 bits (~2 j), 19 car = 35 bits (~1,6 h),
29+ car = plancher 30 bits (~3 min). Les courbes vivent dans
`scone-core::pow_length` (constantes `MAINNET_APEX = 42`,
`MAINNET_FLOOR = 30`, `TESTNET_APEX = 8`, `TESTNET_FLOOR = 2`,
`DEVNET_APEX = 2`, `DEVNET_FLOOR = 1`) ;
les paramètres plats (`tld_pow_difficulty`, etc.) restent dans
`scone-core::network::NetworkParams`. Changer une valeur = changement
de consensus **pour ce réseau**. Invariant : la devnet est une
troisième `NetworkParams` complète et la testnet reste strictement
identique à ce qu'elle était avant l'ajout de la devnet (constantes
Argon2id, courbes, fenêtres d'époque) — la devnet ne partage AUCUNE
constante de consensus avec les autres réseaux au-delà du code
lui-même.

Calibrage mesuré des sommets plats (2026-10, `cargo run --release
-p scone-core --example pow_bench` — débit single-thread ~7,7 M H/s
sur x86-64 de référence) :

| Kind (mainnet) | difficulté | temps réel moyen |
|---|---|---|
| `RegisterTld` | 24 bits | ~7 s |
| `RegisterDomain` | 20 bits | ~54 ms |

À ~65 s/bloc : un claim TLD coûte ~0,1 bloc de CPU mono-cœur, un
domaine ~1 ms de bloc — symbolique pour un utilisateur, coûteux à
industrialiser en masse (anti-squat). À recalibrer si les cibles
matérielles baissent (ARM mobile : ~4-8× plus lent).

Branchements d'état (M8b, livré) : `RegisterTld` exige TOUJOURS son
PoW ; `RegisterDomain` l'exige sur un TLD **ouvert** (le chemin
fermé est assign-only — `AssignDomain`, sans PoW, signé par l'owner
du TLD).

## Règles de validation

Une transaction est valide si et seulement si :

1. **décodage strict** : structure et bornes ci-dessus respectées
   (discriminant connu, version `0x01`, pk 32 o, signature 64 o,
   invariants core re-vérifiés au décodage) ;
2. **binding owner/pk** : `owner == OwnerId::from(public_key)`
   (recomputé) ;
3. **cohérence nom/id** (RegisterDomain) : `domain_id ==
   DomainId(name)` (recomputé) ;
4. **signature** : `verify_strict` Ed25519 de `signature` sur le
   payload signé **recomputé de zéro** avec le network de la chaîne
   (jamais pris d'un pair ni de la transaction telle que fournie) —
   une tx signée pour un autre réseau échoue ici
   (`InvalidSignature`, anti-replay cross-chain) ;
5. **règles d'état** (à l'application dans un bloc — M8b, voir
   `/docs/fr/technical/blockchain.md`) :
   RegisterTld → TLD libre + non ICANN + PoW (difficulté par
   longueur) ; RegisterDomain → TLD enregistré **et ouvert** +
   domaine libre (grâce comprise) + PoW (par longueur) ;
   UpdateDomain → domaine existant, `owner` courant,
   `sequence == current + 1` ;
   TransferTld / RevokeTld / SetTldOpen / AssignDomain → TLD
   existant **et privé**, signataire = owner du TLD ;
   TransferDomain → domaine existant, owner ;
   RenewDomain → domaine existant, owner, extension stricte ≤ 3 ans,
   autorisation PoW (TLD ouvert : même PoW que la création) ou PoS
   (TLD privé : signature du owner du TLD, `proof` vide).

La vérification utilise `verify_strict` (rejet des signatures
malléables et des clés faibles/non canoniques), comme exigé pour des
données non fiables.

## Ordre des opérations dans `push_block`

La validation cryptographique (2 à 4) a lieu pour chaque transaction
**avant** l'application d'état, sur un état de travail atomique par
bloc ; la moindre erreur laisse la chaîne rigoureusement inchangée.

## Assemblage de blocs

`BlockBuilder` (`scone-blockchain`) assemble un bloc depuis une queue
de transactions : `tx_root` recalculé sur la liste ordonnée, header
enchaîné sur le hash parent (`prev_hash`), `version` =
`PROTOCOL_VERSION`. Le payload consensus (payload producteur signé)
et le timestamp
sont fournis par l'appelant ; le trait `Consensus` existant est
utilisé symétriquement à la validation.

## Exemples hex (vecteurs réels, reproductibles — M8b)

### RegisterDomain `example.uip`, timestamp 42, testnet

Payload signé (vecteur épinglé, généré par `scone tx build
register-domain --name example.uip --timestamp 42`). Le `network`
apparaît ici car c'est le PAYLOAD DE SIGNATURE (sa seule place) ; la
transaction wire signée, elle, n'en contient pas :

```text
53434f4e452d54582d5349472d5631       préfixe SCONE-TX-SIG-V1
21                                    disc REGISTER_DOMAIN (0x21)
01                                    version format
0d 73636f6e652d746573746e6574         network (str, "scone-testnet")
0b 6578616d706c652e756970             name (str, longueur 11)
e6a2cbe264f81ed7da4d901c4deec6a3579224b8080fad01cb7461b8c264f181  domain_id
7b2446dadddf640244ac8677a401f148b2b23d1c836d5ad3dd340f8ef2edbed8   owner (clé de build fixe)
2a                                    timestamp (varint, 42)
14 0200000000000000 0000000000000000 04000000
                                      proof (20 o : nonce 2, époque 0, difficulté 4 — testnet)
2a002152f8d19b791d24453242e15f2eab6cb7cffa7b6a5ed30097960e069881db12  pk + début sig
```

### RegisterTld `uip`, timestamp 42

Payload signé (vecteur épinglé) — le `TldId` est le vecteur de
dérivation épinglé dans `/docs/fr/general/naming.md` :

```text
53434f4e452d54582d5349472d5631       préfixe SCONE-TX-SIG-V1
93 01                                 disc REGISTER_TLD + version
0d 73636f6e652d746573746e6574         network (str, "scone-testnet")
03 756970                             tld (str, "uip")
ad5a86d68643d5c22d6a959bb1a315530c77dfda241f1ac1a8107450f3fab25e  tld_id ("uip")
7b2446dadddf640244ac8677a401f148b2b23d1c836d5ad3dd340f8ef2edbed8   owner
2a                                    timestamp (varint, 42)
14 8e00000000000000 0000000000000000 07000000
                                      proof (20 o : nonce 0x8e, époque 0, difficulté 7 — testnet, label de 3 car.)
00                                    open (0x00 = naissance privée)
2a002152f8d19b791d24453242e15f2eab6cb7cffa7b6a5ed30097960e069881db12  pk
```

(Les valeurs complètes sont reproductibles via `scone tx build
register-domain --name example.uip --timestamp 42`,
`scone tx build register-tld --tld uip --timestamp 42` puis
`scone tx sign`. Depuis M8b le PoW est miné automatiquement à la
difficulté testnet — les nonces varient d'un run à l'autre, seuls
network/nom/id/owner/pk sont stables. Depuis le changement de wire
2026-09, l'hex de la transaction signée (`scone tx sign`) ne contient
PLUS les octets `0d 73636f6e652d746573746e6574` : le network ne
voyage que dans le payload à signer.)

## CLI de test et debug

```bash
scone tx build register-domain --name example.uip --timestamp 42  # hex du payload à signer
scone tx build register-tld --tld uip --timestamp 42              # idem, claim de TLD
scone tx sign <payload|tx-hex> --identity alice                   # tx complète signée (hex)
scone tx verify <tx-hex>                                          # validation locale complète
```

`sign` accepte en entrée le payload de `build` ou une transaction
complète ; dans tous les cas l'owner et la clé sont **recomputés**
depuis l'identité du keystore, jamais repris de l'entrée.

## Historique de version

| Version | Contenu | Statut |
|---|---|---|
| 1 | `public_key` + `signature` embarquées, payload `SCONE-TX-SIG-V1`, octet de version `0x01`, `RegisterDomain` porte le nom, `RegisterTld` (0x93), famille M8a : `TransferTld` (0x47), `RevokeTld` (0x6B), `SetTldOpen` (0xB8), `AssignDomain` (0xD4), `RenewDomain` (0x3C) — formats wire normatifs ; M8b : champ `network` signé en tête de chaque payload, `RegisterTld` porte le nom, règles d'état complètes, difficultés PoW par réseau | remplacée |
| 1 (2026-09, wire sans network) | **retrait du champ `network` du wire de toutes les transactions** : le network_id vit uniquement dans le payload de signature (`signing_payload(tx, network)` l'injecte en tête) ; la séparation de réseaux repose sur la signature (`InvalidSignature` sur payload étranger, l'erreur `WrongNetwork` disparaît) ; `TX_FORMAT_VERSION` reste `0x01` (testnet relançable, pas de coexistence) ; **tous les TxId changent** (le TxId est le hash de l'encodage wire signé, qui perd 14-15 octets) ; digest PoW, genèse et data-dirs per-network inchangés | remplacée |
| 1 (2026-10, PoW V2 sel d'époque) | **digest PoW V2 anti pré-minage** : `BLAKE3("SCONE-POW-V2" ‖ network ‖ challenge ‖ epoch_le ‖ nonce_le)` avec `epoch = block_height / 144` du bloc QUI APPLIQUE la tx ; charge utile wire de la preuve 12 → **20 octets** (`nonce_le ‖ epoch_le ‖ difficulty_le`), l'époque déclarée est vérifiée STRICTEMENT contre l'époque du bloc appliquant (rejet typé `ProofEpochMismatch`, aucune tolérance), best-effort `(height+1)/144` à l'admission mempool, drapeau CLI `--epoch` sur les chemins hors-ligne ; changement cassant pré-mainnet : le payload V1 12 octets est rejeté, **tous les TxId changent** (le champ `proof` s'allonge de 8 octets) | courante |

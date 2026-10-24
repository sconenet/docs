# Protocole

Document de référence du protocole Scone : format binaire,
encodage canonique, transactions, enregistrements DNS, blocs,
messages P2P, limites et compatibilité des versions.
L'implémentation de référence est la crate
[`scone-protocol`](../../../crates/scone-protocol).

Les éléments marqués **(provisoire)** ne sont pas définitifs.

## Version du protocole

- constante : `PROTOCOL_VERSION: u32 = 1` (`scone-protocol`) ;
- portée par `Hello.version` (poignée de main) et
  `BlockHeader.version` ;
- règles de compatibilité :
  - version reçue **supérieure** à la version locale → rejet
    (`ProtocolError::UnsupportedVersion`) ; le nœud ne devine
    pas le format d'une version qu'il ne connaît pas ;
  - version `0` → invalide ;
  - il n'y a qu'un seul jeu de formats : les octets de version
    de transaction (`0x01`) et de bloc doivent correspondre
    **exactement** aux valeurs locales ;
- pas de découpage majeur/mineur à ce stade : un seul entier,
  incrémenté à toute rupture du format filaire.

## Principes du format binaire

- binaire, jamais JSON pour les données de protocole ;
- déterministe et canonique : deux objets logiquement égaux
  produisent **exactement les mêmes octets** ;
- analyse séquentielle, sans allocation autant que possible
  (emprunts d'octets) ;
- toute entrée réseau est non fiable : longueurs et compteurs
  sont vérifiés **avant** toute allocation ; le décodage ne
  panique jamais.

### Boutisme

- tous les entiers sont des varints LEB128 (groupes de 7 bits,
  poids faible en premier) : le boutisme natif ne s'applique
  pas ;
- adresses IPv4/IPv6 et identifiants de 32 octets : tableaux
  d'octets bruts, sans préfixe de longueur ;
- il n'y a aucun entier large de taille fixe dans le format :
  une seule règle d'encodage des entiers.

### Varint (LEB128 minimal)

Chaque octet porte 7 bits de données ; le bit de poids fort est
à 1 sur tous les octets sauf le dernier. Un `u64` tient en 1 à
10 octets.

| valeur | octets (hex) |
|---|---|
| 0 | `00` |
| 1 | `01` |
| 127 | `7f` |
| 128 | `80 01` |
| 255 | `ff 01` |
| 16 384 | `80 80 01` |
| 2^32 | `80 80 80 80 10` |
| u64::MAX | `ff ff ff ff ff ff ff ff ff 01` |

L'encodage doit être **minimal** : les formes surdimensionnées
sont rejetées au décodage (`InvalidVarint`). C'est la condition
nécessaire pour qu'une valeur n'ait qu'une seule séquence
d'octets valide (encodage canonique). Rejets : `80 00`
(non minimal), `ff×9 02` (débordement u64), `ff×10` (trop
long).

### Chaînes et octets

- `bytes` : longueur varint + octets bruts ;
- `str` : pareil, avec décodage UTF-8 obligatoire ;
- toute longueur est plafonnée (voir [Limites](#limites)) et
  vérifiée avant lecture.

### Identifiants

| Type | Représentation filaire |
|---|---|
| `DomainId` | 32 octets bruts |
| `TldId` | 32 octets bruts |
| `OwnerId` | 32 octets bruts |
| `PublicKeyRef` | 32 octets bruts |
| `RecordHash` | 32 octets bruts |
| `BlockHash` | 32 octets bruts |
| `MerkleRoot` | 32 octets bruts |
| `DomainName` | `str` (≤ 253 octets, revalidé par `scone-core`) |

Les dérivations (`BLAKE3-256` + préfixes `SCONE-DOMAIN-V1`,
etc.) restent définies par `scone-core` (voir
`/docs/en/general/naming.md`).

## Transactions

```text
Transaction = disc u8 || version u8 || payload
```

| Discriminant | Type |
|---|---|
| `0x21` | `RegisterDomain` |
| `0x2A` | `RegisterTldPow` |
| `0x52` | `UpdateDomain` |
| `0x57` | `RenewTldPow` |
| `0x84` | `RegisterDomainPow` |
| `0x93` | `RegisterTld` |
| `0xC3` | `RenewDomainPow` |

Chaque discriminant est une constante arbitraire opaque (aucune
logique de séquence ; `0x00` jamais valide) — table normative
complète dans `/docs/en/technical/transactions.md`.

L'octet `version` (valeur courante `0x01`) suit le
discriminant : toute autre valeur est explicitement rejetée
(`UnsupportedVersion(value)`). La plupart des transactions sont
signées (Ed25519) — SAUF les quatre formes pow-to-owner sans
signature (`0x2A`, `0x57`, `0x84`, `0xC3`) : la preuve minée EST
l'autorisation, et son challenge lie la clé du bénéficiaire et le
réseau. Spécification complète, charge signée et règles de
validation dans **`/docs/en/technical/transactions.md`** (normatif).

Depuis le changement filaire de 2026-09, **les transactions ne
portent plus de champ `network`** : le `network_id` (`str`
≤ 16) est injecté dans la **charge de signature** (formes
signées) ou le **challenge PoW** (formes pow-to-owner) — une
transaction préparée pour un réseau ne transite jamais vers un
autre (voir `/docs/en/technical/blockchain.md`).

### REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

`RegisterDomain` porte le **nom canonique en clair**, suivi de
son `DomainId` — la cohérence `domain_id == DomainId(name)` est
revérifiée au décodage (recalculée, jamais crue sur parole).
`proof` porte le PoW d'enregistrement (20 octets : nonce +
époque + difficulté — sel d'époque `epoch = height/144` du bloc
d'application, vérifiée STRICTEMENT à l'application contre la
constante réseau et l'époque du bloc ; rejet typé
`ProofEpochMismatch` en cas de discordance — M8b, voir
`transactions.md`). `public_key` est la clé Ed25519 du
signataire (exactement 32 octets) ; `owner` doit être la
dérivation BLAKE3 de cette clé (recalculée, jamais crue) ;
`signature` fait exactement 64 octets.

### REGISTER_TLD (0x93)

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

Revendication d'un TLD (registre distinct du registre de
domaines, M7) : le TLD est **porté en clair** comme le nom d'un
domaine (M8b — le défi PoW dérive du nom), et `tld_id` est la
dérivation `SCONE-TLD-V1` de ce nom, disjointe de tout
`DomainId` par construction. Règles d'état (M8b) : TLD libre +
non réservé ICANN (`IcannTldReserved` — les TLD racine ICANN ne
sont revendicables par personne) + PoW à la difficulté **basée
sur la longueur** de l'étiquette (courbe `pow_length` — voir
`blockchain.md`) ; revendiqué = fermé (assignation seule).

### UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

Invariant revérifié au décodage : `sequence > 0` et `owner` ==
dérivation de `public_key`. Pas de `timestamp` : l'ordre passe
par `sequence`. La transaction ne porte que des références
compactes — le contenu DNS complet reste dans le DHT (voir
ci-dessous).

## Enregistrements DNS (côté DHT)

Les enregistrements DNS complets n'entrent jamais dans la
blockchain : ils sont échangés via le DHT (messages
`GetRecord`/`Record`) et engagés sur chaîne par leur seul
`RecordHash`.

### RecordData

```text
RecordData = type.v || ttl.v || payload(type)
```

Les codes de type sont les **vrais codes DNS** (IANA). Le TTL
(secondes, varint) suit le code de type et couvre tout
l'enregistrement : bornes `MIN_TTL = 1` .. `MAX_TTL =
2 592 000` (30 jours ; défaut `DEFAULT_TTL = 86 400`).

| Code | Type | Charge |
|---|---|---|
| 1 | A | 4 octets |
| 2 | NS | `name` |
| 5 | CNAME | `name` |
| 12 | PTR | `name` |
| 15 | MX | `preference.v`, `name` |
| 16 | TXT | `str` ≤ 4096 |
| 28 | AAAA | 16 octets |
| 33 | SRV | `priority.v`, `weight.v`, `port.v`, `target(name)` |
| 257 | CAA | `flags u8`, `tag` (1–15 o), `value` (`str` ≤ 4096) |
| 32768 | GEOROUTE | GeoRoute (voir ci-dessous) |
| autre | Unknown | code.v + `bytes` ≤ 4096 |

Exemple : `A 192.0.2.1` (TTL 86 400) → `01 80 a3 05 c0 00 02
01` (type, TTL varint, adresse).

Un `Unknown` portant un code connu est rejeté à l'encodage (le
décodage ne peut jamais en produire).

#### GEOROUTE (GeoDNS)

Type privé Scone (plage non-IANA). Une règle de routage
géographique :

```text
GeoRoute  = scope || count.v(≤ 32) || endpoint…
scope     = 0x00                    -- world
          | 0x01 continent.v        -- UN M49
          | 0x02 country.v          -- ISO 3166-1 numeric
          | 0x03 region.v           -- region code
          | 0x04 asn.v              -- AS number
          | 0x05 prefix.addr[16] || prefix.len u8   -- IP prefix (normalized)
endpoint  = target || priority.v || weight.v
target    = 0x00 octets[4]          -- binary IPv4
          | 0x01 octets[16]         -- binary IPv6
```

**Règle canonique** : les points de terminaison d'un scope
donné sont encodés **triés** (priorité croissante, puis poids
décroissant, puis cible croissante) ; le tri est appliqué à
l'encodage et un filaire non trié est rejeté au décodage
(`NonCanonical`), comme pour les jeux d'enregistrements. Les
doublons complets de points de terminaison sont rejetés.

**Validation** : `priority > 0`, `weight ≤ 100` (0 = point de
terminaison inactif), ≤ 32 points de terminaison par scope
(`GEO_MAX_ENDPOINTS`). Un code pays/continent inconnu de la
table locale (`continent_of_country`) est **accepté** au niveau
filaire : la table du résolveur peut être plus à jour que celle
du publieur. Le TTL est celui du `RecordData` parent
(inchangé).

### DnsRecord

```text
domain_id[32] sequence.v expiration.v count.v(≤ 256) records…
```

**Règle canonique** : un jeu d'enregistrements est une
collection non ordonnée. Il est encodé **trié par ordre
lexicographique de l'encodage individuel de chaque
enregistrement**, strictement croissant (doublons rejetés). Un
filaire non trié est rejeté au décodage (`NonCanonical`).
L'ordre du `Vec` d'origine n'est pas transporté.

### SignedDnsRecord

```text
DnsRecord owner[32] signature(bytes ≤ 128)
```

### RecordHash

```text
RecordHash = BLAKE3-256("SCONE-RECORD-V1" || canonical(DnsRecord))
```

Le hachage couvre **le seul contenu de l'enregistrement** ; la
signature du propriétaire est vérifiée séparément sur les mêmes
octets canoniques. *(Décision : l'engagement ne couvre pas la
signature — une version antérieure de ce document mentionnait
le « SignedDnsRecord complet » ; coupler le hachage à la
signature n'apportait rien et liait l'engagement au schéma de
signature.)*

## Blocs

### BlockHeader

```text
version.v height.v prev_hash[32] tx_root[32] timestamp.v consensus(bytes ≤ 65,536)
```

- `version` : version du protocole (règles ci-dessus) ;
- `height` : hauteur (genèse = 0) ;
- `prev_hash` : hachage du bloc précédent (zéros pour le bloc
  de genèse) ;
- `tx_root` : engagement sur la liste des transactions (arbre
  de Merkle défini par `scone-blockchain`, voir
  `/docs/en/technical/blockchain.md`) ;
- `timestamp` : information d'ordre (Unix, secondes) ;
- `consensus` : charge du producteur (`SCONE-BLOCK-V1 ‖ pk[32]
  ‖ sig[64]`, voir `blockchain.md`) — la genèse d'un réseau
  porte le `network_id` brut, et une genèse d'allocation
  (amorçage mainnet) peut porter un manifeste `ALLOC-V1`
  (≤ 60 Kio) ; bornée par `MAX_CONSENSUS_LEN` (64 Kio).

### Block

```text
BlockHeader tx_count.v(≤ 4096) transactions…
```

Analyse séquentielle ; le compteur est plafonné **avant**
allocation ; toute troncature est rejetée. Contrairement aux
jeux d'enregistrements, l'ordre des transactions est
**significatif** (ordre de consensus) et préservé exactement.

`BlockHash` (hachage de l'en-tête canonique) et `tx_root` sont
calculés par `scone-blockchain` sur les encodages définis ici
(formules exactes dans `/docs/en/technical/blockchain.md`).

## Messages P2P

```text
Message = disc u8 || payload
```

Le transport (cadrage, libp2p…) est hors périmètre ; une trame
de transport ne devrait pas dépasser `MAX_MESSAGE_LEN`
(1 Mio).

| Disc | Message | Direction | Charge | Limite | Réponse attendue |
|---|---|---|---|---|---|
| `0x01` | `Hello` | les deux | `version.v` | — | `Hello` |
| `0x02` | `Ping` | les deux | — | — | `Pong` |
| `0x03` | `Pong` | les deux | — | — | — |
| `0x04` | `GetBlock` | les deux | `hash[32]` | — | `Block` |
| `0x05` | `GetBlocks` | les deux | `start.v` `max.v` | `max ≤ 128` | ≤ `max` × `Block` |
| `0x06` | `Block` | les deux | `Block` | tx ≤ 4096 | — |
| `0x07` | `Transaction` | les deux | `Transaction` | — | — |
| `0x08` | `GetRecord` | les deux | `domain_id[32]` | — | `Record` |
| `0x09` | `Record` | les deux | `SignedDnsRecord` | — | — |
| `0x0a` | `Checkpoint` | les deux | `Checkpoint` | signataires ≤ 64 | — |
| `0x0b` | `GetCheckpoints` | les deux | — | — | fenêtre de points de contrôle |
| `0x0c` | `GetStatus` | les deux | — | — | `NodeStatus` |
| `0x0d` | `NodeStatus` | les deux | état du nœud (réseau, pointe, hauteur, NAT, domaines, mempool, pairs, finalité, opérateurs frais) | — | — |
| `0x0e` | `LookupName` | les deux | `name(str)` | — | `NameState` |
| `0x0f` | `NameState` | les deux | état on-chain du nom + TLD | — | — |
| `0x10` | `GetOwnedNames` | les deux | `owner[32]` | — | `OwnedNames` |
| `0x11` | `OwnedNames` | les deux | domaines + TLD vivants du propriétaire | — | — |
| `0x12` | `Error` | réponse | message d'erreur applicatif (`str`) | — | — |

### Point de contrôle (M5)

Un point de contrôle agrégé part sur le fil tel que : étiquette
`SCONE-CKPT-V1` || corps `CheckpointData` (116 o, LE —
identique à la fin de la préimage de signature : `epoch(8)
height(8) block_hash(32) prev_checkpoint_hash(32) state_root(32)
recovery(4)`) || `count.v` || `pk[32] ‖ sig[64]` par
signataire, **triés par pk croissant, sans doublons** `count ≤
64` (`MAX_CHECKPOINT_SIGNERS`). Décodage strict : troncature,
mauvaise étiquette, pk non canonique, doublons ou non-triés =
rejet (`None`), jamais de panic (corruption testée en fuzz).

`GetCheckpoints` demande la fenêtre de finalité ; la réponse
est la fenêtre `checkpoint_window()` (au plus `CHECKPOINT_KEEP`
points de contrôle, du plus ancien au plus récent) via un ou
plusieurs messages `Checkpoint`.

`TxId` est désormais défini par `scone-blockchain`
(`BLAKE3-256("SCONE-TX-V1" || canonical(Transaction))`, voir
`/docs/en/technical/blockchain.md`). Le mempool et
l'anti-rejeu fenêtré (`TxReplay`, `REPLAY_WINDOW_BLOCKS = 256`)
sont livrés dans le relais (M4) ; seul `GetTransaction` reste
différé (sa définition dépend des décisions de consensus
restantes).

## Répartition blockchain / DHT

```text
BLOCKCHAIN                      DHT
──────────                      ───
domain_id                       domain_id
owner                           SignedDnsRecord (full, signed)
sequence
record_hash
```

La chaîne dit *qui* possède et *quel est l'état courant*
(sequence + record_hash). Le DHT fournit *le contenu*. Un
enregistrement DHT est accepté si et seulement si :

1. sa signature est valide pour le propriétaire enregistré sur
   la chaîne ;
2. sa `sequence` est cohérente avec (ou supérieure à) celle de
   la chaîne ;
3. le `RecordHash` recalculé sur son encodage canonique
   correspond au `record_hash` de la chaîne.

## Limites

| Constante | Valeur | Rôle |
|---|---|---|
| `MAX_NAME_LEN` | 253 | longueur d'un nom |
| `MAX_TLD_LEN` | 63 | longueur d'un TLD |
| `MAX_TXT_LEN` | 4096 | chaîne TXT |
| `MAX_PROOF_LEN` | 256 | preuve d'enregistrement |
| `MAX_SIGNATURE_LEN` | 128 | signature (Ed25519 = 64) |
| `MAX_RECORDS_PER_SET` | 256 | enregistrements par jeu |
| `MAX_DHT_RECORD_LEN` | 64 Kio | enregistrement DNS signé encodé (charge DHT : filaire, stockage, gossip) — borne totale vérifiée à l'encodage ET avant tout décodage ; 15–30× au-dessus de tout jeu réaliste (SPF+DKIM+DMARC ≈ 2–5 Kio), 16× sous le plafond de transport. |
| `GEO_MAX_ENDPOINTS` | 32 | points de terminaison par scope GeoRoute |
| `MAX_UNKNOWN_DATA` | 4096 | données d'un type inconnu |
| `MAX_TXS_PER_BLOCK` | 4096 | transactions par bloc |
| `MAX_CONSENSUS_LEN` | 64 Kio | charge de consensus d'un en-tête (élargie pour le manifeste de genèse `ALLOC-V1` — amorçage mainnet) |
| `MAX_BLOCKS_PER_REQUEST` | 128 | blocs par `GetBlocks` |
| `MAX_MESSAGE_LEN` | 1 Mio | trame de transport (recommandé) |

Tout dépassement → `LimitExceeded`, détecté avant allocation :
un pair malveillant ne peut pas provoquer d'allocation
disproportionnée au décodage.

## Encodage canonique

Règle fondamentale :

```text
canonical object → canonical binary encoding → hash / signature
```

- varints minimaux uniquement ;
- jeux d'enregistrements triés strictement croissants ;
- un objet logiquement égal n'a exactement **qu'un** encodage
  valide ;
- le décodage rejette les formes non canoniques
  (`NonCanonical`), il n'existe donc pas d'encodages multiples
  valides pour un même objet.

## Gestion des erreurs

`ProtocolError` (`scone-protocol::error`) :

| Variante | Signification |
|---|---|
| `Truncated` | entrée terminée au milieu d'une valeur |
| `InvalidVarint` | varint non minimal, débordant ou trop long |
| `IntegerOutOfRange` | entier ne tenant pas dans son type cible |
| `UnsupportedVersion` | version nulle ou supérieure à la version locale |
| `UnknownDiscriminant` | discriminant de transaction/message inconnu |
| `LimitExceeded` | longueur ou compte au-delà d'une limite |
| `InvalidUtf8` | chaîne non-UTF-8 |
| `TrailingBytes` | octets restants après une valeur complète |
| `Validation` | violation d'un invariant de `scone-core` |
| `NonCanonical` | encodage valide champ par champ mais non canonique |

Le décodage ne panique jamais sur des données réseau.

## Signatures

- schéma : **Ed25519** (RFC 8032) via `scone-crypto`
  (`ed25519-dalek`, vérification `verify_strict` — signatures
  malléables et clés faibles rejetées) ;
- **transactions de chaîne** : chaque `RegisterDomain` /
  `RegisterTld` / `UpdateDomain` embarque `public_key` (32 o) +
  `signature` (64 o) sur la charge
  `"SCONE-TX-SIG-V1" || canonical_encode(tx_without_signature)` ;
  spécification normative complète :
  `/docs/en/technical/transactions.md` ;
- **enregistrements DHT** (`SignedDnsRecord`) : la charge
  signée est l'encodage canonique du `DnsRecord` ; la
  vérification croise `record + owner + signature` avec
  l'identité enregistrée dans la blockchain ;
- règle transversale : `owner` et tous les
  hachages/engagements sont **recalculés**, jamais repris d'une
  valeur fournie.

## Règles de consensus

Livrées (décrites dans `/docs/en/technical/blockchain.md`) :
producteur de bloc signé (charge `SCONE-BLOCK-V1`, comité élu
par points de contrôle), finalité par points de contrôle agrégés
(`SCONE-CKPT-V1`, choix de fourche au plancher de finalité),
PoW d'enregistrement par longueur de nom. Sujets restants :
frais anti-spam fins, rotation de la clé du propriétaire.

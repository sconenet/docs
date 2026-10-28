# Stockage — persistance locale du nœud (redb)

> Document normatif. Toute divergence entre ce document et
> `crates/scone-storage/src/` est un bug (identifier la source de
> vérité, la corriger). Référence d'implémentation : crate
> `scone-storage` (format v1), backend `redb` 4.2.

## Objectif

La persistance locale embarquée du nœud : blocs canoniques, états de
domaines, cache DHT, métadonnées. Le store est une **persistance**,
jamais une autorité : l'état RAM ([`ChainState`], la source du
consensus) est reconstruit au démarrage depuis le store, et chaque
écriture est un **delta** (domaines modifiés par un bloc), jamais une
copie complète de l'état.

Contrainte transversale (côté utilisateur, chaque jalon) : **empreinte
mémoire minimale**. Les gros volumes (blocs, états de domaines, cache
DHT) vivent dans redb et sont accédés via des curseurs/intervalles —
jamais chargés entièrement en RAM, jamais dupliqués inutilement.

## Frontière : le trait `NodeStore`

`crates/scone-storage/src/lib.rs` définit la seule face de stockage
visible du reste du code (blockchain, futur relais M4) :

- `append_block(height, hash, bytes)` /
  `append_block_with_state(..., deltas)` — ajout atomique (voir
  ci-dessous) ;
- `block_at_height(h)` / `block_by_hash(hash)` — lectures ponctuelles ;
- `tip() -> (height, hash)` — O(1), lu depuis `meta` ;
- `put_domain_state` / `domain_state` / `domain_count` — compteur
  **maintenu** (`meta["domain_count"]`), jamais un
  `COUNT(*)` ;
- `iterate_domains(after, max) -> DomainPage` — pagination à base de
  curseur (lots de 100 par défaut), ordre croissant des octets du
  `DomainId` ;
- `put_tld_state` / `tld_state` / `tld_count` / `iterate_tlds` —
  mêmes contrats pour le registre TLD (`tld_count` maintenu,
  curseur borné) (M7d) ;
- `put_dht_cache` / `dht_cache` — octets opaques du `SignedDnsRecord` ;
- `resolve_name(canonical)` / `resolve_tld(tld)` — index Nom→Id
  persistant (P0.2), lecture ponctuelle O(1), aucune itération sur les
  domaines (défaut `None` = backend sans index, l'appelant retombe sur
  `iterate_domains` ou l'état de la chaîne) ;
- `tld_floors_all` / `snapshot_tld_floors_all` — lecture des
  planchers de durée de vie des TLDs ouverts (owner-option : tables
  dédiées, `tld_floors_v1` vivante et `snapshot_v1_floors` figée) ;
- `rewind_to_height(h)` — réorg : retire les blocs au-dessus de `h` et
  ramène `tip`/`tip_height` à `h` en UNE transaction
  (préliminaire à `persist_state_snapshot`, voir ci-dessous) ;
- `meta_get` / `meta_set` ;
- `snapshot_meta` / `snapshot_domains` / `snapshot_tlds` — lecture du
  snapshot d'état de démarrage (M6b ; défauts vides = backend sans
  snapshot, `load_chain` retombe alors sur le chargement complet).

Règles :

- **Aucun type redb dans l'API** : des octets (`&[u8]`, `[u8; 32]`) et
  des types core uniquement. Le backend est remplaçable.
- **Sans async, pas de mutabilité interne** : les écritures prennent
  `&mut self` (redb sérialise les écrivains), les lectures prennent
  `&self` (une courte transaction de lecture par appel).
- L'API ne revalide jamais les blocs : le hachage passé à
  `append_block*` est le hachage **recalculé** fourni par l'appelant
  (`Blockchain::push_block`).

## Schéma des tables redb

| Table | Clé (tri) | Valeur |
|---|---|---|
| `blocks_by_height` | hauteur `u64` (b-tree ordonné) | `hash(32) ‖ height(8 BE) ‖ bloc canonique` |
| `blocks_by_hash` | hachage du bloc (32 B) | encodage canonique du bloc |
| `domains` | `DomainId` (32 B, ordre des octets) | `DomainStateBytes` (ci-dessous) |
| `tlds` | `TldId` (32 B, ordre des octets) | `TldStateBytes` (ci-dessous, M7d) |
| `dht_cache` | `DomainId` (32 B) | octets du `SignedDnsRecord` (opaque) |
| `names_v3` (P0.2) | clé d'index (32 B, ordre des octets) | id cible (32 B) |
| `name_reverse_v3` (P0.2) | id cible (32 B, ordre des octets) | `tag(1) ‖ nom canonique` |
| `meta` | `&[u8]` | `&[u8]` |
| `snapshot_v1_meta` (M6b) | `u64` = 0 (slot unique) | `height(8 BE) ‖ tip_hash(32)` = exactement 40 B |
| `snapshot_v1_domains` (M6b) | `DomainId` (32 B, ordre des octets) | `DomainStateBytes` (même format que `domains`) |
| `snapshot_v1_tlds` (M6b) | `TldId` (32 B, ordre des octets) | `TldStateBytes` (même format que `tlds`) |
| `tld_floors_v1` (owner-option) | `TldId` (32 B, ordre des octets) | `u64` (plancher d'expiration, BE implicite redb) |
| `snapshot_v1_floors` (M6b) | `TldId` (32 B, ordre des octets) | `u64` — copie figée des planchers (état AU snapshot, pas au sommet) |

Clé `meta` supplémentaire (P0.3/P0.4) :

| Clé | Valeur |
|---|---|
| `verified_snapshot_v1` (P0.3) | `height(8 BE) ‖ tip_hash(32) ‖ manifest_hash(32)` = 72 B — marqueur d'un snapshot importé VÉRIFIÉ (réservé, comme `tip`) |

Clés `meta` :

| Clé | Valeur |
|---|---|
| `format_version` | `u64 BE` = `STORAGE_FORMAT_VERSION` (1) |
| `tip` | hachage (32 B) du bloc sommet |
| `tip_height` | `u64 BE` (rend `tip()` O(1)) |
| `domain_count` | `u64 BE` (compteur maintenu) |
| `tld_count` | `u64 BE` (compteur maintenu, M7d) |

L'en-tête `hash ‖ height` de `blocks_by_height` rend chaque index
auto-contenu (vérifiable sans lire le bloc) ; les deux index sont mis à
jour dans la même transaction.

### Convention de genèse

Le bloc de genèse est déterministe (constantes de protocole) et n'est
**jamais stocké** en octets. Un store frais a `tip == (0,
genesis_hash)` ; le premier bloc ajouté a la hauteur 1. Le hachage de
genèse est toujours re-dérivé
(`scone_blockchain::genesis_hash()`), jamais lu.

## Format d'encodage `DomainStateBytes` (M8b)

65 ou 97 octets selon le tag de version de tête, entiers big-endian à
largeur fixe (canonique, comparable octet par octet) :

```text
DomainStateBytes = tag ‖ owner(32) ‖ sequence(8 BE) ‖ registered_at(8 BE)
                   ‖ valid_until(8 BE) ‖ mutations(8 BE) [‖ record_hash(32)]

tag = 0x01 : record_hash présent → exactement 97 octets
tag = 0x02 : record_hash absent  → exactement 65 octets
```

- `registered_at` / `valid_until` (M8b) : l'expiration de
  l'enregistrement (1 an au claim, renouvelable, voir
  `/docs/en/technical/blockchain.md`).
- `mutations` (V3) : le compteur de mutations monotone du namespace
  (génération CAS anti-replay — voir `transactions.md`, Le compteur
  de mutations). Les encodages M8b (57/89 octets, sans compteur)
  sont **illégitimes** : décodage strict → `Corrupted` (reset
  testnet ; un store pré-compteur se régénère par rejeu).

Les encodages M7 (41/73 octets, sans expiration) sont **illégitimes**
depuis M8b : décodage strict → `Corrupted`. (Un store M7 se régénère
par replay : `load_chain_replay`.)

- `owner` : `OwnerId` (32 B) ;
- `sequence` : la dernière séquence `UpdateDomain` appliquée (0 juste
  après un `RegisterDomain`) ;
- `record_hash` : l'engagement on-chain courant de l'ensemble
  d'enregistrements DNS (`None` tant qu'aucun `UpdateDomain` n'a été
  appliqué).

Décodage **strict** (`DomainStateBytes::decode`) : tag inconnu,
longueur erronée ou octets excédentaires → `StorageError::Corrupted`,
jamais de panic.

## Format d'encodage `TldStateBytes` (owner-option)

Longueur variable selon la variante d'énum — 34 octets pour un TLD
**privé**, 2 pour un TLD **ouvert** (ni séquence ni hachage
d'enregistrement) :

```text
TldStateBytes = tag(0x04) ‖ flag(0x00|0x01) ‖ owner(32)? ‖ until(8)
```

- `flag` (owner-option, énum `TldState`) : `0x00` =
  `TldState::Open` (à personne — libre-service + PoW, aucun octet de
  propriétaire), `0x01` = `TldState::Private` (32 octets de
  propriétaire + 8 octets de compteur suivent). Strictement
  canonique — tout autre octet → `Corrupted`. L'invariant « ouvert ⟺
  pas de propriétaire » est STRUCTUREL : les deux états ne peuvent
  pas être confondus.
- **`until` (8 octets, BE64)** : l'horloge propre du TLD — la fin du
  terme privé (claim/renouvellement + 3 ans) ou l'échéancier de mort
  ouvert (plancher 1 an à la naissance, poussé par chaque domaine
  vivant). C'est de nouveau un attribut de l'entrée (changement de
  cycle de vie TLD). La table dérivée `tld_floors_v1` (+ sa copie
  figée `snapshot_v1_floors`) suit toujours les planchers,
  reconstruits au replay et restaurés au démarrage — un nœud redémarré
  tue les mêmes TLDs qu'un nœud continu.

Les encodages M7d (33 octets, tag `0x01`), M8b (34 octets, tag `0x02`),
les layouts pré-compteur tag `0x04` (42/2 octets, sans compteur), le
tag legacy `0x03` (`until` embarqué) sont
**illégitimes** : décodage strict → `Corrupted` (pré-mainnet, la
régénération par replay est le chemin de migration).

Le tag n'existe que pour l'évolution du format : une future mise en
page incrémente le tag et les vieux lecteurs échouent avec `Corrupted`
au lieu de deviner. Décodage **strict** (`TldStateBytes::decode`) :
tag inconnu, longueur erronée, octets excédentaires ou flag non
canonique → `StorageError::Corrupted`, jamais de panic.

## Garanties d'atomicité

`append_block_with_state` écrit dans **une seule**
`WriteTransaction` redb :

1. le bloc dans `blocks_by_height` (contrôle préalable : la hauteur ne
   doit pas exister avec un contenu différent, sinon
   `StorageError::Conflict`) ;
2. le bloc dans `blocks_by_hash` ;
3. chaque delta d'état de domaine (`domains`) ;
4. le compteur `domain_count` (incrémenté du nombre de nouveaux
   domaines **uniquement** ; un `UpdateDomain` d'un domaine existant
   ne l'incrémente pas) ;
5. chaque delta du registre TLD (`tlds`, M8b : claims, transferts,
   ouverture/fermeture) — **dans la même transaction** : un bloc écrit
   sans ses changements TLD est structurellement impossible ;
6. le compteur `tld_count` (nouveaux TLDs uniquement) ;
7. chaque **retrait** (M8b) : domaines expirés par le GC déterministe
   (`removed_domains`) et TLDs révoqués (`removed_tlds`) quittent le
   store dans la même transaction, compteurs décrémentés — un
   redémarrage ne peut jamais ressusciter un enregistrement expiré ;
8. les deltas de l'**index de noms** de P0.2 (upserts et retraits, voir
   ci-dessous) — mêmes garanties : un crash ne laisse jamais une entrée
   d'index pointant vers un domaine manquant ;
9. `tip` et `tip_height`.

redb commit à la manière d'un WAL : un crash en pleine écriture laisse
l'état cohérent **précédent**. **Un bloc écrit sans son état est donc
structurellement impossible** — c'est l'invariant clé du jalon.

Garanties supplémentaires :

- **Monotonie** : ajout à une hauteur ≠ `tip_height + 1` →
  `StorageError::NonMonotonicHeight`, rien n'est écrit (vérifié
  *avant* l'ouverture de la transaction d'écriture) ;
- **Idempotence** : ré-ajouter le bloc sommet courant avec le même
  couple `(height, hash)` est un no-op (le chemin rapide compare
  hauteur et hachage ; à hauteur égale, des octets de bloc identiques
  sont aussi un no-op silencieux, des octets différents un `Conflict`) ;
- **Base corrompue** : fichier tronqué/pollué → erreur typée
  (`StorageError::Database`/`Corrupted`) à l'ouverture ou en lecture,
  jamais de panic (testé) ;
- **Version de format** : un `format_version` ≠ 1 →
  `StorageError::UnsupportedFormat` (un chemin de migration explicite
  est exigé, jamais deviner).

## Intégration blockchain (`integration.rs`)

Séparation des rôles : [`ChainState`] en RAM = autorité de consensus ;
store = persistance. La stratégie retenue (la plus simple, comme
documenté) : **persister l'état à chaque bloc** — `store_block` écrit
le bloc + les états des domaines **touchés par ce bloc** (delta). Le
replay complet reste possible et sert de chemin de contrôle/réparation.

- `store_block(store, chain, block, hash)` : encode le bloc
  canoniquement, lit l'état final des domaines touchés ET des TLDs
  claimés depuis l'état RAM (déjà mis à jour par `push_block`), ajout
  atomique.
- `load_chain(store)` : **sans replay** — lit le sommet + le bloc
  sommet (lectures de bloc O(1)), restaure l'état des domaines par
  pages de 100 (`DOMAIN_PAGE`) **et le registre TLD par pages de 100**
  (`TLD_PAGE`, M7d — fermeture de la fenêtre de redémarrage : sans
  lui, un nœud redémarré oubliait chaque TLD claimé et rejetait chaque
  `RegisterDomain` sous ce TLD avec `UnknownTld`, alors même que son
  disque détenait l'état complet). La chaîne RAM ne garde que genèse +
  sommet ; les blocs historiques restent dans le store et sont servis
  depuis lui. Depuis M8, `load_chain` reconstruit ensuite l'**index
  anti-replay TXID** (voir ci-dessous).
- `load_chain_replay(store)` : rejoue tous les blocs stockés un par un
  (jamais tous en RAM) avec validation complète ; le sommet résultant
  doit égaler le sommet stocké, sinon `Corrupted`.
- `rebuild_replay_index_from_store(store, chain, tip_height)`
  (M8) : rescanne les derniers
  `min(REPLAY_WINDOW_BLOCKS, tip_height)` blocs persistés (un seul
  bloc en RAM à la fois) et réinsère chaque TxId dans l'index de la
  chaîne **à sa hauteur d'inclusion d'origine**, via l'API publique
  `Blockchain::rebuild_replay_index` (pas d'accès aux champs
  privés). Bloc manquant sous le sommet ou indécodable dans la fenêtre
  → `Corrupted` (ces octets ont déjà servi à construire la chaîne : un
  échec ici signale un store endommagé).
- `touched_domains(block)` : ids des domaines modifiés par les
  transactions du bloc, dédupliqués, ordre du bloc.
- `touched_tlds(block)` : ids des TLDs mutés par le bloc
  (`RegisterTld`, `TransferTld`, `RevokeTld`, `SetTldOpen`),
  dédupliqués, ordre du bloc (M8b).
- `store_block_with_removals(...)` : variante de `store_block`
  recevant aussi les domaines retirés par le GC du bloc (M8b) pour
  l'écriture atomique décrite ci-dessus. Le relais alimente cette
  liste depuis `Blockchain::push_block_with_gc` (le résultat
  d'application expose `gc_removed_domains`). **Éviction GC du cache
  DHT** : les entrées `dht_cache` des domaines retirés sont supprimées
  dans la MÊME transaction que le bloc (les deltas d'ajout portent la
  table `dht_cache`) — un enregistrement DHT ne survit jamais au
  domaine qu'il servait, les entrées expirées ne s'accumulent jamais
  indéfiniment (test : `gc_evicts_the_dht_cache_entry`).

Le relais (M4) utilisera : `load_chain` au démarrage, puis pour chaque
bloc accepté `push_block` (RAM) puis `store_block` (disque).

## Persistance des réorgs (`rewind_to_height` + `persist_state_snapshot`)

Une réorg (bascule vers une branche sœur plus forte) ne peut pas passer
par l'ajout monotone — écrire le suffixe gagnant échouerait avec
`NonMonotonicHeight`. Le chemin dédié (correctif minorité-collante,
relais + store) :

1. `rewind_to_height(fork_height)` — UNE transaction redb retire les
   blocs au-dessus du point de fork et ramène `tip`/`tip_height`
   (l'index de noms et les états vivants sont corrigés à l'étape
   suivante) ;
2. `persist_state_snapshot(store, chain)` — réécrit les tables
   vivantes (`domains`, `tlds`, index de noms, planchers TLD) depuis
   l'export de l'état RAM post-réorg : les tables vivantes ne décrivent
   JAMAIS la branche PERDANTE au démarrage ;
3. ré-ajout monotone du suffixe adopté (deltas ordinaires).

Un échec de persistance sur ce chemin est journalisé et dégradé (le
store guérit au démarrage par replay) — jamais fatal.

## Snapshot d'état de démarrage (M6b)

`load_chain` tel que décrit ci-dessus reste encore O(taille de l'état) :
aucun bloc n'est rejoué, mais toute la table `domains` est relue.
C'est le chemin état-persisté-à-chaque-bloc qui rend le replay
inutile. Le snapshot M6b adresse un coût différent : le cas où l'état
RAM doit être reconstruit **par replay** (réparation, contrôle, ou un
backend sans état persisté par bloc). Il offre un démarrage en
**O(sommet − H)** — rejouer seulement le suffixe au-dessus du snapshot.

### Écriture

- **Intervalle** : `SNAPSHOT_INTERVAL = 64` (constante publique de
  `scone-storage`). Le déclencheur est l'ajout du bloc
  `k·SNAPSHOT_INTERVAL + 1` : cette transaction fige l'état des tables
  vivantes **tel qu'il est avant l'application de ce bloc**, c'est-à-dire
  l'état canonique après le bloc `k·SNAPSHOT_INTERVAL`.
- **Jamais au sommet** : le snapshot se situe donc à `tip − 1` au plus
  près, toujours strictement sous le sommet au moment de son écriture.
  Un crash juste après ne peut jamais laisser un snapshot « au-dessus »
  de la chaîne persistée.
- **Atomique** : le gel (vidage + copie des tables `domains`/`tlds`
  vers `snapshot_v1_*`, écriture de `snapshot_v1_meta`) se produit dans
  la MÊME transaction redb que l'ajout du bloc déclencheur — un crash
  laisse soit l'ancien snapshot intact, soit le nouveau complet avec
  son bloc (WAL redb).
- **Slot unique** : chaque snapshot REMPLACE le précédent (tables
  vidées puis recopiées) — pas d'accumulation, pas d'élagage à gérer,
  l'empreinte disque est bornée par ~2× l'état.
- Le format des états figés est strictement celui des
  `DomainStateBytes`/`TldStateBytes` (réutilisation, pas de nouvel
  encodage) ; seule la ligne `snapshot_v1_meta` est un nouveau format
  (`height(8 BE) ‖ tip_hash(32)`, décodage strict : toute autre
  longueur → `Corrupted`).

### Lecture au démarrage (`load_chain`)

1. Le sommet est validé comme toujours (hachage recalculé depuis
   l'en-tête du bloc sommet vs `meta["tip"]`).
2. Si `snapshot_meta()` existe, sa hauteur `H` satisfait
   `0 < H < tip_height`, et le hachage RECALCULÉ du bloc stocké à `H`
   égale le `tip_hash` du snapshot (**ancre** — un snapshot étranger à
   la chaîne persistée, altéré ou venant d'un autre fichier est détecté
   ici), l'état figé est restauré par pages de 100 puis seuls les
   blocs `H+1..=tip` sont rejoués (validation complète, identique aux
   blocs vivants).
3. Le sommet atteint par le replay doit égaler le sommet stocké.
4. **Tout échec sur ce chemin (snapshot manquant, hauteur inutilisable,
   ancre invalide, octets indécodables, replay échoué) dégrade
   silencieusement vers le chargement complet pré-M6b** (lecture des
   tables vivantes, sans replay) — un mauvais snapshot ne fait jamais
   échouer le démarrage. La compatibilité rétroactive est le cas
   particulier « pas de snapshot » : un store écrit avant M6b (sans
   les tables `snapshot_v1_*`) se charge à l'identique,
   `snapshot_meta()` étant `None`.

Le suffixe rejoué est borné par `SNAPSHOT_INTERVAL` (au plus 64 blocs
au moment du déclenchement + les blocs minés depuis) : le démarrage
est O(sommet − H) avec H garanti récent.

## Index anti-replay TXID reconstruit au démarrage (M8)

L'index anti-replay (`included`, TxId → hauteur d'inclusion, fenêtre
`REPLAY_WINDOW_BLOCKS = 256` — voir
`/docs/en/technical/blockchain.md`) est une **fonction pure de la
chaîne canonique** : il n'est jamais persisté. Comme `load_chain` ne
rejoue pas l'historique, une chaîne restaurée démarrait avant M8 avec
un index **vide** — un nœud redémarré acceptait la ré-inclusion de
n'importe quelle transaction encore dans sa fenêtre de replay
(limite documentée ; les règles d'état restaient la seule sauvegarde).

M8 ferme cette fenêtre : après restauration (chemin complet OU
snapshot + replay du suffixe), `load_chain` appelle
`rebuild_replay_index_from_store`, qui relit les derniers
`min(REPLAY_WINDOW_BLOCKS, tip_height)` blocs persistés — un bloc en
RAM à la fois, borné mémoire comme partout — et réinsère chaque TxId à
sa **hauteur d'origine**. Un store de moins de 256 blocs reconstruit
donc tout son index.

Propriétés :

- **Décisions identiques au nœud vivant** : mêmes entrées, même règle
  d'élagage (`h > tip − 256`) — un nœud qui redémarre et un nœud qui
  ne s'est jamais arrêté rejettent les mêmes replays (testé : comptage
  exact de l'index et égalité avec l'index vivant élagué).
- **Sémantique d'élagage inchangée** : une transaction plus vieille que
  la fenêtre reste admise par l'index (et échoue alors sur les règles
  d'état le cas échéant) — exactement comme avant.
- **Le snapshot ne fige PAS l'index** : le replay du suffixe du
  snapshot reconstruit les entrées au-dessus de H, le rescan M8 ajoute
  la partie de la fenêtre SOUS H (même contrat que owner_keys/grâce :
  reconstruit, jamais persisté).
- **Coût** : ≤ 256 lectures ponctuelles de blocs au démarrage,
  O(fenêtre) — indépendant de l'âge de la chaîne.
- Un bloc manquant ou indécodable **dans la fenêtre** fait échouer le
  démarrage avec `Corrupted` (ces octets ont déjà servi à construire la
  chaîne) ; hors fenêtre, les blocs historiques sont intouchés.

`load_chain_replay` (confiance zéro) reconstruit l'index de la même
manière via son replay complet — les deux chemins convergent.

## Index Nom → Id persistant (P0.2)

`DomainId = BLAKE3-256(DOMAIN_ID_VERSION ‖ nom canonique)` n'est pas
inversible : résoudre un nom (relais/DNS) exige un index persistant.
P0.2 l'ajoute au store, avec la même discipline transactionnelle que
le reste.

### Schéma

Deux tables compagnes, écrites et effacées ensemble :

```text
names_v3        key  = BLAKE3-256("SCONE-NAME-IDX-V1" ‖ nom canonique)  (32 B)
                val  = id cible (32 B : DomainId ou TldId)

name_reverse_v3 key  = id cible (32 B)
                val  = tag(0x01 domaine | 0x02 TLD) ‖ nom canonique
```

- La clé `names_v3` est un **hachage du nom, pas le nom en clair**
  (tables d'index compactes, pas de gossip de noms via les dumps). La
  clé est une **fonction pure du nom** : l'index est reconstructible à
  tout moment depuis la chaîne (`rebuild_name_index`), aucun
  métadonnée supplémentaire n'est persistée pour lui.
- `name_reverse_v3` stocke le nom (exactement une fois) parce que les
  chemins de retrait (GC des domaines expirés, `RevokeTld`) ne
  connaissent que des **ids** : il faut bien retrouver la clé hachée à
  effacer. Le tag distingue les deux espaces de noms.
- Séparation stricte : `resolve_name` ne répond JAMAIS une entrée TLD
  et `resolve_tld` jamais une entrée domaine (le tag de la table
  inverse décide ; un `DomainName` a ≥ 2 labels, un TLD en a 1 — les
  deux dérivations de clé partagent l'espace `names_v3` mais pas les
  résultats).
- Le `DomainId` reste 32 octets, jamais réduit (règle dure du projet :
  identité de consensus, résistance aux collisions).

### Cohérence transactionnelle

Les upserts (`RegisterDomain`, `AssignDomain`, `RegisterTld` — noms
portés en clair par les transactions) et les retraits (GC,
révocations — ids nus) sont portés par le `StateDelta` du bloc
(`name_upserts` / `name_removals`) et appliqués dans la **même
transaction redb** que les écritures de bloc et d'état. Un crash ne
peut donc laisser ni une entrée d'index pointant vers un domaine
manquant (pas de fantômes), ni un domaine enregistré sans son entrée.
Un ajout échoué n'écrit rien (testé). Les variantes nommées
`put_domain_state_with_name` / `put_tld_state_with_name` /
`remove_domain_state` offrent la même atomicité aux outils de
réparation.

`store_block` / `store_block_with_removals` dérivent les deltas de
noms des MÊMES transactions du bloc — pas de paramètre supplémentaire à
l'interface du relais.

### Reconstruction (`RedbStore::rebuild_name_index`)

L'index est une fonction pure de la chaîne canonique, comme l'index
anti-replay M8. `rebuild_name_index` le reconstruit depuis les
**blocs persistés** (les `DomainStateBytes` ne contiennent PAS le nom
— seulement propriétaire/séquence/expiration — donc la source de
vérité est le flux de transactions, où `RegisterDomain` et
`AssignDomain` portent le nom canonique en clair) :

1. passe 1 : scan `blocks_by_height` genèse→sommet, un bloc en RAM à
   la fois ; collecte des liens `nom → id` vus dans
   `RegisterDomain`/`AssignDomain`/`RegisterTld` ;
2. passe 2 : filtrage aux seules cibles **vivantes** (l'état persisté
   fait autorité) — un domaine enregistré puis GC'd/expiré n'est pas
   réindexé ;
3. passe 3 : effacement + réécriture complète des deux tables en UNE
   transaction (crash → ancien ou nouvel index complet).

Un bloc manquant ou indécodable → `Corrupted` (mêmes octets qui ont
construit la chaîne). Le coût est O(noms distincts vus) en RAM (≤ 253
B par nom) et O(blocs) en lectures — un outil de réparation, pas un
chemin de démarrage.

### Ce que l'index n'est PAS

- Pas une autorité : l'état RAM de la chaîne reste la seule autorité ;
  l'index est un cache reconstructible (une divergence est un bug de
  transactionnalité, détectable par `rebuild_name_index` comparé à
  lui-même).
- Pas figé dans le snapshot M6b : les tables `names_v3`/
  `name_reverse_v3` sont vivantes et suivent les deltas de chaque
  bloc ; le snapshot d'état ne les embarque pas (reconstruction par
  deltas à la reprise, ou `rebuild_name_index` en réparation).

### Ce que le snapshot n'engage PAS

L'état RAM `ChainState` porte des structures internes non engagées par
le consensus (clés de propriétaires, fenêtres de grâce). Elles ne sont
PAS figées : le replay du suffixe les reconstruit en appliquant les
transactions (déterministe, même contenu que le replay complet pour la
fenêtre concernée). L'index anti-replay TXID suit le même contrat (M8,
voir section précédente) : reconstruit au démarrage, jamais figé.
L'égalité bit-exacte avec `load_chain_replay` est vérifiée par test sur
l'état canonique (state_root + SMT).

## Snapshot d'état VÉRIFIÉ — morceaux + manifeste + vérification cryptographique (P0.3)

Le snapshot M6b ci-dessus accélère le démarrage d'un nœud qui fait
confiance à son disque. P0.3 adresse un besoin différent : transporter
un état entre nœuds (amorçage d'un nouveau nœud, réparation) sans
faire confiance au transport. Le format et la vérification vivent dans
`crates/scone-storage/src/verified_snapshot.rs` + la reconstruction de
racine dans `crates/scone-blockchain/src/snapshot_verify.rs`.

### Modèle de confiance

Le snapshot porte l'état d'un bloc **finalisé** — un point de contrôle
signé par le quorum du comité (`CheckpointData.state_root`, la racine
O(1) `state_root_smt`). L'authenticité du point de contrôle (quorum de
signatures Ed25519) est validée par la couche chaîne AVANT import.
L'import lui-même vérifie l'adhérence snapshot ↔ point de contrôle par
RECALCUL :

1. **hachage de chaque page** : `blake3("SCONE-SNAP-PAGE-HASH" ‖
   encodage complet de la page)` doit égaler le hachage annoncé par le
   manifeste (un octet falsifié → rejet) ;
2. **somme des entrées == compteurs du manifeste** (une page manquante
   ou un compteur erroné → rejet) ;
3. **racine SMT recalculée depuis les entrées importées ==
   `checkpoint.data.state_root`** — LA vérification cryptographique :
   tout l'état (domaines + TLDs) est replié dans un SMT neuf
   (`snapshot_verify::recompute_state_root`, feuilles
   `SCONE-LEAF-DOM-V1`/`SCONE-LEAF-TLD-V1` exactement comme le chemin
   vivant) et la racine doit égaler celle signée par le comité. Un
   snapshot falsifié, tronqué ou incomplet est rejeté AVANT TOUTE
   ÉCRITURE ;
4. **cohérences structurelles** : `height`/`tip_hash`/`state_root` du
   manifeste == ceux du point de contrôle ; ids strictement croissants
   (pas de doublons) ; index de pages séquentiels ; bornes DoS
   (≤ 10 000 entrées/page, ≤ 2^20 pages, budget total).

Échec de vérification → `StorageError::SnapshotRejected`, le store
reste strictement inchangé (tout est vérifié avant la transaction
d'écriture).

### Format sérialisé (strict, big-endian, décodage sans pitié)

```text
manifest = "SCONE-SNAP-MAN-V1"
        ‖ height(8) ‖ tip_hash(32) ‖ state_root(32)
        ‖ page_count(4) ‖ domain_count(8) ‖ tld_count(8)
        ‖ page_hashes(page_count × 32)

page    = "SCONE-SNAP-PAGE-V1"
        ‖ index(4) ‖ height(8) ‖ state_root(32)      ← chaque page est
        ‖ entry_count(4) ‖ entries                   auto-identifiée

entry   = kind(0x01 domaine | 0x02 TLD | 0x03 plancher TLD) ‖ id(32)
          ‖ [len(2) ‖ state | floor(8 BE)]
          (state = DomainStateBytes 57/89 B ou TldStateBytes 2/34 B —
           les formats canoniques de stockage existants, réutilisés ; le
           kind 0x03 (plancher de durée de vie d'un TLD ouvert,
           owner-option) a un format dédié SANS préfixe de longueur :
           id(32) ‖ floor(8 BE), taille fixe)
```

Ordre des pages : toutes les entrées de domaine (ids croissants),
puis toutes les entrées TLD (ids croissants), puis les planchers de
durée de vie des TLDs FIGÉS à la hauteur du snapshot
(`snapshot_tld_floors_all` — pas la table vivante) — l'ordre de la
pagination par curseur du store. Chaque page porte `(index, height,
state_root du point de contrôle, entries)` : une page seule sait à
quel snapshot elle appartient. Pages de 256 entrées à l'export
(`VERIFIED_SNAPSHOT_PAGE_ENTRIES`) ; bornes DoS à l'import. Les
planchers TLD (`tld_floors_v1`) font partie du format portable : le
kind 0x03 les porte, et l'import les exige strictement croissants et
ciblant un TLD PRÉSENT dans le snapshot — sinon l'importateur ne
tuerait jamais ce qu'un exportateur tuerait (fork silencieux, rejeté).

### Export (`export_verified_snapshot(store, checkpoint)`)

Sérialise l'état du DERNIER POINT DE CONTRÔLE FINALISÉ — pas le
sommet. Émet les domaines, les TLDs, puis les planchers de durée de
vie des TLDs figés à la hauteur du snapshot (symétrie export/import :
l'export omettait historiquement les planchers — corrigé, test de
régression `export_carries_tld_lifetime_floors`). Prérequis : le store
détient un snapshot M6b persisté (`snapshot_v1_*`) exactement à la
hauteur du point de contrôle (la fenêtre
`CHECKPOINT_KEEP`/`SNAPSHOT_INTERVAL` garantit qu'un tel état est
régulièrement disponible) ; l'ancre du bloc est RECALCULÉE depuis les
octets stockés et doit égaler `checkpoint.data.block_hash`. Sinon →
`StorageError::SnapshotUnavailable` (l'export est une action
explicite : attendre le prochain intervalle/point de contrôle).

### Import (`import_verified_snapshot(store, checkpoint, manifest, pages)`)

Vérifie tout (voir ci-dessus), puis écrit en UNE transaction redb :
tables vivantes `domains`/`tlds` + compteurs + `tip = (H, hachage du
point de contrôle)`, tables figées `snapshot_v1_*` ancrées au même
bloc, et le marqueur réservé
`meta["verified_snapshot_v1"]` (`height ‖ tip_hash ‖
manifest_hash`). Cible : store vide uniquement (sinon
`SnapshotUnavailable`).

### Coût de vérification (documenté)

Reconstruction SMT par insertions ordonnées : N entrées × O(40)
hachages ≈ O(N·40). ~2,1 M hachages blake3 pour 100 000 domaines
(dizaines de ms). PONCTUEL — un import d'amorçage/réparation, jamais un
chemin par bloc : acceptable. L'insertion ordonnée (ids croissants)
crée chaque point de branche une seule fois ; de toute façon la racine
ne dépend que du contenu (garantie SMT, fuzz différentiel dans
`smt.rs`).

## Amorçage d'état — sans replay de genèse (P0.4)

Un nouveau nœud rejoint le réseau sans rejouer l'historique depuis la
genèse :

```text
point de contrôle finalisé (relais, validé par la couche chaîne)
        │
        ▼
snapshot vérifié (manifeste + pages, vérification cryptographique ci-dessus)
        │  import → état à H + sommet (H, hachage) + marqueur
        ▼
bloc d'ancrage H (demandé au réseau PAR HACHAGE — append_bootstrap_anchor)
        │
        ▼
blocs récents H+1..H+k (relais, hauteur croissante, ajout atomique
        │               ordinaire — l'API de stockage n'exige que ça)
        ▼
synchronisé — le démarrage rejoue SEULEMENT le suffixe au-dessus de H
```

- **API de stockage exposée au relais** : `append_bootstrap_anchor(H,
  hash, bytes)` (pose le bloc d'ancrage : marqueur exigé, cohérence
  hauteur/hachage avec le marqueur, idempotent) puis ajout atomique
  ordinaire par hauteur croissante. La récupération réseau (par
  hachage depuis le manifeste) concerne le relais, pas le store.
- **`load_chain`** reconnaît le marqueur : l'historique sous H est
  légitimement absent (nœud non-archive). Concrètement : le rescan de
  la fenêtre anti-replay M8 est plancheré à H (les blocs sous H
  n'existent pas et ne manquent pas) ; un sommet sans bloc stocké
  APRÈS un import (ancre pas encore posée) → diagnostic typé
  `SnapshotUnavailable` (« l'ancre doit venir du réseau »), pas une
  corruption.
- **Nœud archive vs nœud état-complet** : un nœud archive (drapeau
  opérateur) garde tout — il sert les blocs historiques au réseau et
  démarre via les chemins existants. Un nœud état-complet (défaut P0.4
  pour un nouveau nœud CONFIGURÉ ainsi) démarre à H : il valide les
  blocs futurs (replay du suffixe) mais ne peut pas servir
  l'historique sous H ni rejouer la genèse. **Le comportement par
  défaut ne change PAS** : ce chemin n'est actif que si l'opérateur
  déclenche explicitement l'import (`import_verified_snapshot` est une
  action dédiée, jamais automatique) ; chaque nœud existant démarre à
  l'identique.
- **Égalité avec le nœud complet** : après avoir rejoint le suffixe
  H+1..H+k, l'état, le sommet et le `state_root_smt` sont bit-exact
  avec un nœud complet au même sommet (testé) ; l'index anti-replay
  couvre exactement les txs à/au-dessus de H — un nœud non-archive ne
  peut pas connaître la fenêtre sous H, les règles d'état restent la
  sauvegarde (une tx dupliquée sous H échoue sur l'état, jamais
  acceptée deux fois).

## Politique mémoire

- **Curseurs partout** : `iterate_domains` borne la page à `max`
  entrées (100 par défaut) via le `range` paresseux de redb ; toute la
  table des domaines n'est jamais matérialisée. Les grandes listes
  (TLDs, domaines) passent obligatoirement par ce curseur
  (`iterate_tlds`, même contrat, M7d).
- **Deltas uniquement** : `append_block_with_state` n'écrit que les
  domaines et TLDs modifiés par le bloc — jamais l'état complet ;
- **Pas de duplication synchrone** : l'état RAM n'est pas
  re-persisté par bloc ; l'état disque n'est pas rechargé par
  bloc ;
- **Lectures ponctuelles** : chaque lecture ouvre une courte
  transaction de lecture, retourne la valeur, ferme ; les
  `AccessGuard`s redb ne vivent que le temps de la copie bornée ;
- **Blocs historiques jamais en RAM** : `load_chain` ne charge que le
  sommet ; `load_chain_replay` streame hauteur par hauteur ;
- **Compteur maintenu** : `domain_count` est O(1), pas un scan.

## Bornes DoS

Le store borne ce qu'un pair hostile peut lui faire persister ou lui
faire allouer :

- **`MAX_DHT_CACHE_ENTRY` (64 KiB)** : un enregistrement DHT plus grand
  est rejeté avant toute transaction d'écriture par `put_dht_cache` →
  `StorageError::TooLarge` (rien n'est écrit) ;
- **`MAX_DOMAIN_PAGE` (10 000)** : `iterate_domains` borne
  *internement* la page demandée à `min(max, 10_000)` — un `max`
  excessif ne matérialise jamais plus de 10 000 états en RAM ;
  `iterate_tlds` a la même borne (`MAX_TLD_PAGE`, M7d) ;
- **Clés `meta` réservées** : `meta_set` rejette `tip`,
  `tip_height`, `format_version`, `domain_count`, `tld_count` et
  `verified_snapshot_v1` (P0.3) avec
  `StorageError::ReservedKey` — écraser la comptabilité interne du
  store le corromprait silencieusement.
- **Cache DHT doublement borné (LRU)** : `DHT_CACHE_MAX_ENTRIES`
  (100 000 entrées) ET `DHT_CACHE_MAX_BYTES` (2 GiB — garde cas
  hostile ; chaque entrée est déjà plafonnée à 64 KiB par
  `MAX_DHT_CACHE_ENTRY`), la première borne atteinte évince par ordre
  de clé. Les compteurs (entrées + octets) sont maintenus à l'écriture
  — jamais un scan pour compter ; la reconstruction exacte n'a lieu
  qu'à l'ouverture (le seul scan complet, au démarrage). Accesseurs
  `dht_cache_entries`/`dht_cache_bytes`, bornes réglables via
  `set_dht_cache_bounds` (test :
  `dht_cache_respects_the_double_bound`).

## Modèle de confiance

Deux chemins de chargement, deux niveaux de garantie :

- **`load_chain` — confiance au disque local** : le hachage du sommet
  est **recalculé** depuis l'en-tête du bloc sommet et comparé au
  `meta["tip"]` stocké (divergence → `StorageError::Corrupted`,
  jamais une chaîne restaurée sur un hachage forgé), et les états de
  domaines sont strictement décodés. Cependant, les blocs historiques
  ne sont **pas rejoués** : les états persistés sont pris à la lettre.
  C'est le démarrage rapide d'un nœud qui fait confiance à son propre
  disque (écrit uniquement par ce même nœud, atomiquement). Depuis
  M6b, ce chemin utilise d'abord le **snapshot d'état** quand il est
  exploitable (voir ci-dessous) : l'ancre du snapshot est recalculée et
  le suffixe au-dessus de H est revalidé comme des blocs vivants — la
  partie rejouée est ainsi confiance-zéro, la base figée confiance-disque.
- **`load_chain_replay` — confiance zéro** : tous les blocs sont relus
  et revalidés un par un (validation complète, identique aux blocs
  vivants) ; le sommet résultant doit égaler le sommet stocké. C'est
  l'outil de contrôle/réparation.

**Recommandation** : un nœud strict (ou un nœud dont le fichier de
stockage a pu être altéré — disque partagé, restauration d'une sauvegarde
douteuse) doit démarrer via `load_chain_replay`, pas `load_chain`.

## Concurrence

redb 4.x n'autorise **qu'un seul objet `Database` par fichier et par
processus** (la double `open` simultanée échoue avec
`DatabaseAlreadyOpen`). Le modèle pris en charge : un `RedbStore` cloné
(`Arc<Database>`) partagé entre threads — écrivains sérialisés par
`&mut self` + le verrou interne de redb, lecteurs concurrents
(transactions de lecture snapshot). Après le drop complet de tous les
handles, le fichier peut être rouvert.

## Tests (`tests/redb_store.rs` + modules `#[cfg(test)]`)

Chaque test utilise son propre tmpfile redb (`tempfile`). Couverture :

- aller-retour de blocs genèse→N, redémarrage, relecture, replay ;
- M7d : registre TLD persisté par delta atomique, restauré au
  redémarrage (un `RegisterDomain` sous un TLD claimé avant le
  redémarrage reste admissible — fenêtre fermée), compteurs exacts à la
  réouverture, curseur `tlds` croissant, un ajout échoué ne laisse
  aucun TLD, clé réservée `tld_count` rejetée ;
- atomicité : un ajout échoué ne laisse rien (sommet, domaines, index
  intacts) ; deltas en doublon comptés une fois ;
- curseurs de domaines : pagination 100/7, ordre croissant de
  `DomainId`, départ strictement après un id existant ou manquant,
  `max = 0` ;
- compteur `domain_count` exact (insertion/mise à jour/réouverture) ;
- aller-retour cache DHT + écrasement ;
- concurrence : lectures depuis deux threads sur des handles clonés ;
- base corrompue : fichier tronqué / poubelle → erreur typée, pas de
  panic (`catch_unwind`) ;
- M6b : snapshot écrit à l'intervalle, strictement sous le sommet ;
  démarrage depuis snapshot = replay complet bit-exact (state_root +
  SMT, état logique complet, sommet) ; snapshot altéré (tip_hash
  corrompu, état indécodable, hauteur > sommet, meta de mauvaise
  longueur) → repli silencieux vers le chargement complet, même chaîne
  finale ; suffixe rejoué < intervalle ; remplacement du snapshot (pas
  d'accumulation) au franchissement de l'intervalle ; store pré-M6b
  sans tables de snapshot se charge à l'identique ;
- M8 : index anti-replay reconstruit au démarrage — une tx ré-incluse
  dans la fenêtre est rejetée (`TxReplay`) après restauration ; une tx
  plus vieille que la fenêtre est admise (élagage inchangé, les règles
  d'état décident) ; comptage exact == index vivant (==
  `load_chain_replay`) pour un store plus court et plus long que la
  fenêtre ; démarrage snapshot : la partie de la fenêtre sous H est
  reconstruite par le rescan, celle au-dessus par le replay du suffixe ;
- P0.2 : index de noms — `resolve_name == DomainId::from_name` pour
  120 domaines (vivant, après redémarrage, après effacement +
  reconstruction), `resolve_tld` pour le TLD claimé ; le GC d'un
  domaine expiré retire l'entrée (pas de fantômes), avant et après
  reconstruction ; séparation stricte des espaces de noms domaine/TLD ;
  un ajout échoué ne laisse aucune entrée d'index ; la clé d'index est
  la dérivation BLAKE3 documentée ;
- P0.3/P0.4 (`tests/verified_snapshot.rs`) : export → import →
  démarrage == replay complet (état, sommet, `state_root_smt`
  bit-exact ; l'état importé est celui du bloc finalisé, pas le
  sommet) ; page falsifiée d'un octet → rejet avant toute écriture
  (store strictement inchangé) ; page manquante → rejet ;
  `state_root` erroné (manifeste ↔ point de contrôle, puis racine
  recalculée ≠ racine signée) → rejet ; compteurs / hauteurs erronés /
  sommets incohérents / ids en doublon → rejet ; l'import exige un
  store vide ; l'export exige un snapshot persisté à la hauteur du
  point de contrôle ; contrat d'ancre (`append_bootstrap_anchor` :
  marqueur exigé, cohérence, idempotence) ; format sérialisé strict
  (octets excédentaires, tags, bornes DoS) ; snapshot à H puis blocs
  H+1..H+k rejoints → chaîne identique au nœud complet
  (état/sommet/racine), démarrage sans historique sous H, index
  anti-replay = exactement les txs à/au-dessus de H.

[`ChainState`]: ../../../crates/scone-blockchain/src/state.rs

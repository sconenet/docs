# Modèle de sécurité — sélection du comité et anti-grinding

> Document normatif. Toute divergence doc/code = bug.

## Vue d'ensemble

Le comité de l'époque N+1 est tiré depuis la seed
`next_seed(signing_hash(checkpoint N)) = blake3("SCONE-SEED-V1" || signing_hash(checkpoint N))`,
sur le **pool d'éligibilité figé au bloc finalisé par le
point de contrôle N**. `signing_hash` est le hachage du **contenu** du
point de contrôle (époque, hauteur, bloc, prev, racine d'état — ensemble
de signatures exclu) : la seed est une fonction pure de CE qui a été
finalisé, jamais de QUEL sous-ensemble de quorum l'a signé — deux vues
détenant des sous-sembles distincts (tous deux valides) du même point de
contrôle élisent le même comité.

## Menace : grinder la seed du comité

Un attaquant veut influencer la composition du comité. Trois fenêtres
d'attaque existent ; les trois sont structurellement fermées :

### 1. Avant la seed — impossible de grinder sa position

La seed de l'époque N+1 dérive du **hachage du point de contrôle N**,
signé par quorum (floor(2n/3)+1 du comité N — `quorum_for`). Le point de
contrôle N engage le
`state_root` d'un état qui **fige le pool** (base PoS = état exact au
bloc finalisé, `FinalizedBase.pool`). Produire une seed différente
exigerait un point de contrôle différent signé par quorum — c'est-à-dire
corrompre ≥ 1/3 du comité N, ce qui est l'hypothèse de faute exclue par
le modèle BFT.

### 2. Après la seed — trop tard pour rejoindre

Toute inscription (claim de domaine/TLD, apparition d'une nouvelle clé de
propriétaire) ne compte que pour les époques suivantes, dont les seeds ne
sont pas encore connues. Le pool est figé dans `FinalizedBase` AVANT que
quiconque (y compris les signataires du point de contrôle N) puisse
calculer la seed de N+1.

### 3. Sélection « meilleur parmi m » sans avantage

La position d'une clé dans le tirage =
`blake3("SCONE-COMMITTEE-V1" || seed || pk)` (uniforme), dont le seul
degré de liberté est `pk`. Un grinder qui génère m clés et
n'enregistre que la mieux placée doit d'abord **connaître la
seed** — mais à ce moment le pool est déjà figé.
Enregistrer m candidats « au cas où » coûte m preuves de travail
(une par domaine/TLD) et rapporte exactement m/|pool| de présence
attendue : **aucun avantage sur m enregistrements directs**. La
stratégie adaptative (voir la seed, puis choisir quoi enregistrer) est
temporellement impossible.

### 4. Biais du dernier producteur (attaque par rétention)

Le producteur du dernier bloc avant un point de contrôle pourrait
retenir son bloc pour influencer `signing_hash(checkpoint N)` et donc la
seed. Deux garde-fous :

- le point de contrôle signe un état à la hauteur H, et le **pool est
  évalué à l'horodatage du bloc H** — retenir un bloc retarde le point
  de contrôle mais ne change pas le pool d'une époque déjà figée par le
  point de contrôle N−1 (les pools se recouvrent peu d'une époque à
  l'autre) ;
- la rétention coûte à l'attaquant sa propre récompense de production et
  n'affecte que la VIVACITÉ (point de contrôle retardé), jamais la
  SÛRETÉ : deux points de contrôle finaux incompatibles exigeraient
  ≥ 1/3 double signatures, prouvables par `SlashTx` et punies par bannissement.

Conclusion : une VRF ou un commit-reveal n'apporterait rien ici — la
seed est déjà fonction d'un événement signé par quorum, imprévisible-retardable-mais-infalsifiable, et le gel du pool
retire le seul levier adaptatif. Ne pas compliquer le protocole
(contraite P1.7 : « si nécessaire » — analyse : pas nécessaire).

## Équivoque et slashing

Une ancre qui signe deux points de contrôle conflictuels à la même époque
peut être dénoncée par quiconque détient les deux signatures (`SlashTx`,
discriminant `0x15`) : bannissement du pool PoS (journalisé,
réversible sur réorg). Les domaines/TLDs de l'accusé ne sont pas saisis
(sanction minimale — décision M9).

## Acquisition passive et éligibilité PoS

**Question** : un tiers peut-il m'imposer une possession — et donc une
obligation PoS — sans mon consentement ?

**Oui pour la possession, non pour la dette.** Un nom peut être acquis
passivement : les transactions de cession (`TransferTld`,
`TransferDomain`) sont signées par le **cédant** seul (le bénéficiaire
n'est qu'un `OwnerId` opaque, jamais consulté), `AssignDomain` est signée
par le propriétaire du TLD, et les allocations de genèse atterrissent sur
la clé du manifeste. C'est une feature assumée : dons, dead-drops, legs.
L'anti-Sybil n'en dépend pas : un siège par **propriétaire distinct de nom
vivant** (`owner_refcounts`), et chaque possession coûte du travail à
quelqu'un.

**Aucune dette imposable** : le slashing ne punit que l'**équivoque
prouvée** (deux signatures conflictuelles, `SlashTx`, bannissement) ; une
clé passive ne signe rien et ne peut donc jamais être slashée. L'absence
d'un membre tiré au sort va au **recovery** (re-tirage), sans sanction.
Un siège reçu sans l'avoir voulu reste abandonnable : expiration + GC,
`SetTldOpen` (le siège est libéré à l'ouverture), `RevokeTld`, ou
re-transfert.

**Décision tracée — pas de consentement d'acquisition** : un claim en
deux phases (transfert en attente, acceptation signée par le
bénéficiaire) a été considéré et rejeté — il casse les dead-drops et le
« first applied wins », ajoute une machine à états sur les noms, et
protège d'un risque démontrablement inexistant (aucune dette imposable,
cf. ci-dessus).

## Propriétés vérifiées par les tests

- `checkpoint_hash_engages_signatures` : le hachage du point de contrôle
  couvre les signatures (pas de falsification a posteriori) ;
- `seed_chain_and_recovery` : la chaîne de seeds dérive exactement,
  récupération k incluse ;
- `small_committee_never_finalizes` : garde BFT — un comité sous
  `MIN_FINALITY_COMMITTEE_SIZE` ne finalise jamais ;
- e2e 4 relais : un point de contrôle finalise et se propage, même époque
  sur tous les nœuds.

## Attaques par rejeu sur l'autorité déléguée (bindings de compteur)

Cinq transactions mutent QUI peut agir sur un namespace :
`TransferTld`, `TransferDomain` (propriété), `SetRecordDelegates`
(clés déléguées de records), `SetTldOperator` / `SetDomainOperator`
(délégués d'exploitation). Une transaction enregistrée de ces
saveurs reste cryptographiquement valide POUR TOUJOURS (la signature
couvre des champs statiques ; le PoW de cession résout un challenge
dérivé du namespace lui-même). La seule chose qui vieillit est la
fenêtre de rejeu bornée (256 derniers blocs). Au-delà, analyse de
ré-armement :

- **Cession A→B→A** : A transfère à B, B re-transfère à A — l'état
  est revenu à une configuration antérieure. Rejouer la transaction
  A→B ORIGINALE ré-exécute le transfert sans consentement (le check
  propriétaire repasse ; une preuve à challenge statique se résout
  encore).
- **Cycle de délégués** : délégués posés à `{9,10}`, vidés vers
  `∅`, puis N'IMPORTE QUELLE transaction restaurant `{9,10}` — le
  rejeu du premier `SetRecordDelegates` ressuscite des clés
  révoquées.

Un binding calculé comme fonction pure de l'état COURANT
(propriétaire, contenu du jeu, ...) retrouve sa valeur passée
exactement quand l'état y revient — il ré-armerait le rejeu qu'il
doit tuer. La défense est une génération MONOTONE : chaque namespace
porte `mutations : u64`, incrémenté par chacune des cinq
transactions gardées ; une transaction se lie au compteur de la
génération pour laquelle elle a été construite
(`BLAKE3-256("SCONE-BINDING-MUTATIONS-V1" ‖ id ‖ generation ‖
counter)`) et l'application vérifie ce binding contre le compteur
COURANT. Les compteurs ne reviennent jamais ; donc aucun rejeu ne
se ré-arme. Le marqueur de génération (`registered_at` pour les
domaines, `valid_until` pour les TLD) maintient disjointes les vies
précédentes d'un namespace re-enregistré.

Résidu documenté : le binding ne dépend pas de la CLÉ du signataire.
Les cinq transactions gardées sont signées par le propriétaire
ACTUEL du namespace, donc un rejeu inter-générations exige en plus
que le même propriétaire détienne le namespace aux deux générations
et que le compteur retrouve la valeur enregistrée — scénario
auto-contrôlé pour délégués/opérateurs, et cas victime-propriétaire
pour les cessions qui exige que la victime ait re-claimé son propre
namespace mort des années plus tard. Accepté pré-mainnet ; à
réévaluer si le volume de transferts ou le recyclage de namespaces
rend la chose matérielle.

## Limites connues (documentées, non corrigées)

- Le pool éligible inclut les propriétaires de TLDs vivants (claim = PoW
  dépensée) — surface Sybil bornée par le coût PoW par clé ;
- Le `state_root` du point de contrôle n'est vérifié en RAM que lorsque
  le point de contrôle cible le sommet ; le replay rétroactif est un
  chemin stockage/relais (voir `docs/technical/blockchain.md`).

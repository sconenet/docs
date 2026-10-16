# Nommage

## Alphabet

Lettres ASCII minuscules, chiffres et tirets (LDH, RFC 1123) :
`[a-z0-9-]`, avec le tiret ni en première ni en dernière position
d'un label. Pas de majuscules, pas de tiret bas.

La validation est **stricte** pour l'ASCII : un nom non canonique est
rejeté, jamais corrigé. `"ABC"` → erreur (pas de mise en minuscules
silencieuse). Justification : une frontière de protocole doit être
déterministe ; la canonicalisation implicite crée des ambiguïtés
d'identité.

## Noms internationalisés (IDNA)

Les noms non latins sont acceptés en **entrée** et encodés en
punycode (IDNA, RFC 5890/5891, UTS#46) : `例子.uip` a
`xn--fsqu00a.uip` comme forme canonique.

- la forme canonique (on-chain, ids BLAKE3, DNS, stockage) est
  toujours l'**A-label** ASCII : toutes les règles LDH ci-dessus
  s'appliquent à cette forme ;
- la forme unicode d'origine (**U-label**) est conservée en mémoire
  pour l'affichage (`DomainName::unicode()` / `TldName::unicode()`) ;
- la limite de 63 octets par label et la limite totale de 253 octets
  s'appliquent à la forme canonique A-label, pas au UTF-8 brut ;
- l'ASCII reste sur le chemin strict : l'IDNA ne s'applique qu'aux
  entrées non ASCII (`"ABC"` reste invalide) ;
- politique anti-homoglyphes (scripts mixtes, caractères
  confondables) : pas de filtrage pour l'instant, UTS#46 uniquement.

## TLD

Un TLD est un label DNS comme un autre : la limite générique des
labels (RFC 1035 : 63 octets) s'applique, pas une limite spécifique
aux TLD plus courte.

- motif : `[a-z0-9-]{1,63}` (LDH, sans tiret initial/final), sur la
  forme canonique A-label
- limite codée : `TldName::MAX_LEN = 63` octets
- valides : `uip`, `com`, `test`, `abc12`, `0x`, `a-b`, un TLD
  d'exactement 63 octets, `éxemple` (→ `xn--exemple-2ua`), `例子`
  (→ `xn--fsqu00a`)
- invalides : un TLD dont l'A-label dépasse 63 octets, `ABC`, `-ab`,
  `ab-`, chaîne vide

## Domaines

- forme : `label[.label]*.tld` — au moins deux labels (`name.tld`) ;
- chaque label : LDH `[a-z0-9-]{1,63}`, tiret ni initial ni final ;
- longueur totale : ≤ 253 octets ;
- pas de labels vides (`a..b`, `.a`, `a.` invalides) ;
- le dernier label doit être un TLD valide.

### Sous-domaines

Le type `DomainName` accepte toute la hiérarchie dès le départ :

```text
example.uip
shop.example.uip
api.shop.example.uip
```

Un sous-domaine a un `DomainId` **distinct** de celui de son parent.
Les règles d'allocation des sous-domaines (droits du propriétaire du
parent, etc.) sont un travail futur.

## Canonicalisation

Un nom validé est déjà canonique : `canonical() == as_str()`. Sont
donc rejetés aujourd'hui : majuscules, ponctuation autre que le
séparateur `.`, espaces, unicode, caractères invisibles, etc.

## DomainId

```text
DomainId = BLAKE3-256("SCONE-DOMAIN-V1" || canonical_name)
```

- déterministe : même nom canonique ⇒ même id, partout, pour
  toujours ;
- injectif en pratique : deux noms distincts ⇒ deux ids distincts ;
- le nom brut ne doit **jamais** servir d'identifiant interne (clés
  blockchain, clés DHT) — uniquement pour l'entrée et l'affichage ;
- le préfixe `SCONE-DOMAIN-V1` isole les domaines de dérivation :
  changer la règle (alphabet, normalisation) revient à créer un
  nouvel espace de noms versionné (un nouveau préfixe de dérivation),
  sans réécrire l'existant.

### Représentation textuelle

Encodage hexadécimal **en minuscules** (`0-9a-f`) des 32 octets
bruts : exactement 64 caractères, sans préfixe (`0x`), jamais
tronqué.

```text
DomainId([0xab; 32]) → "ababababababababababababababababababababababababababababababababab"
DomainId([0x00; 32]) → "0000000000000000000000000000000000000000000000000000000000000000"
```

Cette forme est produite par `Display` (le `Debug` Rust affiche
`DomainId(<hex>)`, même chaîne). Toute sortie consommée par un humain
ou une machine hors du format binaire canonique — journaux, messages
d'erreur, sortie CLI — DOIT utiliser cette représentation et aucune
autre (pas de troncature, pas de préfixe, pas de majuscules).

## Limites et règles futures (non arrêtées)

- noms internationalisés (IDN) via punycode ;
- longueur minimale des labels de deuxième niveau ;
- expiration/renouvellement des revendications de domaines.

## TLDId (M7a)

Un TLD enregistrable porte sa propre identité, distincte de celle de
n'importe quel domaine :

```text
TldId = BLAKE3-256("SCONE-TLD-V1" || tld)
```

- le préfixe `SCONE-TLD-V1` est **distinct** de `SCONE-DOMAIN-V1` :
  les espaces d'ids TLD et domaine sont disjoints par construction —
  enregistrer le TLD `uip` (une revendication sur l'espace de noms
  `*.uip`) ne peut jamais squatter l'identité d'un domaine, ni
  l'inverse ;
- mêmes règles d'affichage que `DomainId` : hexadécimal minuscule,
  64 caractères, jamais tronqué ;
- vecteur épinglé (dérivation épinglée par test) :
  `TldId("uip") = ad5a86d68643d5c22d6a959bb1a315530c77dfda241f1ac1a8107450f3fab25e`.

La transaction `RegisterTld` (revendication signée d'un TLD, `owner`
toujours recalculé depuis la clé embarquée) fait partie de l'enum
`Transaction` et du format wire depuis M7b : discriminant `0x93`
(constante opaque arbitraire), charge signée `SCONE-TX-SIG-V1`,
règles complètes dans `/docs/en/technical/transactions.md`. Le
`RegisterDomain` d'un domaine, lui, porte le nom canonique en clair.

## Réservation/gouvernance des TLD

Ouvert par M7a : `RegisterTld` revendique un TLD libre au bénéfice de
l'identité dérivée de la clé signataire. Depuis M8b la gouvernance
est complète : revendication avec PoW (difficulté par réseau) ouvrant
un mandat renouvelable de 3 ans (`RenewTld`), espace de noms
**fermé** par défaut (assign-only), ouverture explicite
(`SetTldOpen`), transfert (`TransferTld`), abandon (`RevokeTld`,
re-claimable). Les TLD privés vivent 3 ans (grâce de 60 jours,
gestion gelée pendant la grâce) ; les TLD ouverts portent un plancher
d'un an maintenu par leurs domaines ; les domaines expirent après
1 an (renouvelable, grâce de 30 jours — voir
`/docs/en/technical/blockchain.md`).

**TLD racine ICANN** : réservés. `RegisterTld` sur un TLD de la liste
IANA épinglée (`is_icann_tld`, ~1 440 entrées + RFC 2606) est
rejeté avec une erreur typée (`IcannTldReserved`) — la racine
historique appartient au DNS existant. Au moment de la résolution,
ces noms sont transmis aux serveurs amont configurés
(`--dns-upstream`) ou REFUSED sans amont. Les TLD ajoutés par ICANN
plus tard restent claimables (liste épinglée : cohérence >
fraîcheur).


## Économie du cycle de vie — qui paie quoi, quand

Le modèle d'autorisation suit la nature de la ressource : le travail
(PoW) paie ce qui entre dans le commun, une signature (PoS) gouverne
ce qui appartient à quelqu'un. Résumé normatif :

### TLD privé

- **Naissance — PoW** : revendiqué par preuve de travail, difficulté
  selon la longueur (mainnet : 1 caractère ≈ 34 j de calcul ;
  plancher ≈ 4,3 j à 10+ caractères). Le demandeur devient
  propriétaire et détient un **siège PoS** (pool d'éligibilité des
  validateurs — battements de cœur exigés pour rester frais,
  délégation d'exploitation disponible).
- **Vie — PoS** : la signature du propriétaire autorise tout dans
  l'espace de noms : `assign` (accorder un domaine), `set-tld-open`
  (irréversible — **libère le siège**), `transfer`, `revoke`, et le
  renouvellement des domaines qui vivent dessous.
- **Mandat — 3 ans** : la revendication ouvre un mandat FIXE de
  3 ans (`TLD_TERM_SECS`). **Renouvellement — PoW égal à la
  création** (`RenewTld`, même courbe de difficulté par longueur ;
  défi lié à l'expiration courante — anti-réutilisation) : chaque
  mandat de vie est payé en travail.
- **Expiration — grâce de 60 jours, gestion gelée** : passé
  l'échéance, une grâce de 60 jours (`TLD_GRACE_SECS`) garde le
  renouvellement possible, mais toute opération de gestion autre que
  le renouvellement est gelée — notamment `assign` : un propriétaire
  expiré perd ses droits de gestion jusqu'au renouvellement. Les
  domaines déjà attribués en dessous vivent leur propre vie (leur
  propre `valid_until`) — le TLD gèle, pas ses enfants.
- **Mort — GC** : mandat + grâce écoulés sans renouvellement → le
  TLD disparaît (siège PoS libéré, re-claimable par quiconque avec
  un PoW frais). Un `TransferTld` transporte le mandat restant (pas
  de remise à zéro).

### TLD ouvert (né ouvert)

- **Naissance — PoW réduit** (un tiers moins cher que privé : une
  ressource commune que personne ne gouverne). **Pas de siège
  PoS**.
- **Plancher — 1 an** (`OPEN_TLD_FLOOR_SECS`) : à la naissance (ou à
  l'ouverture via `SetTldOpen`), un plancher d'un an empêche la
  création-destruction instantanée d'un TLD sans domaine.
- **Vie** : personne ne signe rien — les domaines s'enregistrent en
  libre-service. Chaque domaine vivant en dessous repousse
  l'horloge de mort (elle suit l'expiration du domaine le plus
  récent, bornée par le plancher).
- **Mort — GC** quand plus aucun domaine ne vit en dessous et que
  plancher/expiration sont passés ; le nom redevient claimable.

### Domaine sous un TLD ouvert

- **Naissance — PoW** (courbe réduite vs TLD : mainnet 1 caractère ≈
  8,5 j, plancher ≈ une demi-journée).
- **Renouvellement — PoW identique à la naissance** (même courbe de
  difficulté ; le défi est lié à l'expiration courante, donc
  distinct de l'enregistrement et de chaque renouvellement
  antérieur) : chaque mandat de vie est payé en travail, un domaine
  ne peut pas être « récolté » gratuitement (≤ 3 ans d'avance).
- **Mort** : expiration → grâce de 30 j (renouvelable) → GC → le nom
  devient claimable par **n'importe qui** avec un PoW frais.

### Domaine sous un TLD privé

- **Naissance — le PoS du propriétaire du TLD** : seul le
  propriétaire (ou son opérateur délégué) accorde (`assign`) — la
  signature est l'autorité, pas de PoW.
- **Renouvellement — le PoS du propriétaire du TLD** : signé par le
  propriétaire du TLD, champ `proof` vide.
- **Mort** : grâce puis GC ; la re-réclamation repasse par
  l'`assign` du propriétaire.

### Invariants transversaux

- Chaque PoW est salé par **époque**, avec une fenêtre de validité
  **rétrograde** proportionnelle à la difficulté (un minage de
  plusieurs semaines aboutit quand même même si l'époque a tourné).
- L'équivoque du signataire PoS est **slashed** (bannissement à vie
  — M9).
- Chaque opération re-vérifie tout au moment de l'application :
  rien n'est pris sur parole (hashs, preuves et signatures
  recalculés).

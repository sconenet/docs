# CLI (`scone`) — journalisation et verbosité

> Comportement normatif de la journalisation du binaire `scone`.
> Toute divergence doc/code = bug (corriger la source de vérité).

## Principes

- **stdout est réservé à la sortie des commandes** (`scone status`,
  `identity list`, `status`, …). Un script peut rediriger stdout
  sans jamais y trouver de journaux — le pipelining reste fiable.
  Exception admise : `scone relay` émet les lignes de données
  `p2p: <multiaddr>`, `dns: <addr>` et `metrics: <addr>` sur
  stdout dès que les écoutes sont liées
  (données exploitables pour `--bootstrap` / la configuration,
  pas des journaux ; émises exactement une fois chacune,
  immédiatement flushées).
- **Tous les journaux vont sur stderr** (format `tracing` compact),
  y compris ceux du relais — le relais écrivait déjà sur stderr
  avant la migration.
- Les messages d'erreur destinés à l'utilisateur
  (`scone: <erreur>`) restent sur stderr : c'est de l'UX, pas
  des journaux. Une commande en échec imprime aussi une seconde
  ligne sur stderr, `[Failed: <erreur>]`, en écho aux lignes de
  statut des commandes qui réussissent.

## Niveaux et `--verbose` / `-v`

| Invocation | Niveau effectif |
|---|---|
| `scone <cmd>` (sans `-v`) | `warn` |
| `scone relay` (sans `-v`) | `info` |
| `scone -v <cmd>` / `scone relay -v` | `info` |
| `scone -vv …` | `debug` |
| `scone -vvv …` (et plus) | `trace` |

`-v`/`--verbose` est un drapeau **global et répétable** : il est
accepté avant ou après la sous-commande (`scone -vv identity list`
et `scone identity list -vv` sont équivalents).

**Pourquoi `scone relay` reste à INFO par défaut** : le relais est
un démon — il doit journaliser son activité (écoute p2p,
blocs acceptés et produits, pairs connectés, rejets) pour être
opérable, sans exiger un drapeau. Les commandes à usage unique,
en revanche, restent silencieuses (WARN) pour ne pas polluer un
terminal ou un script.

## `RUST_LOG` (surcharge)

Si la variable d'environnement `RUST_LOG` est définie, elle
**remplace entièrement** le niveau dérivé de `-v` (syntaxe
`env-filter`) :

```bash
RUST_LOG=debug scone identity list        # debug without -v
RUST_LOG=error scone -vv relay            # error wins over -vv
RUST_LOG=scone_network=trace,info scone relay   # per-target
```

## Configuration par environnement (`.env`)

Avant l'analyse des arguments, `scone` charge un fichier `.env`
du répertoire courant (`dotenvy::dotenv()` dans `main.rs`).
Fichier absent = non-op silencieuse, jamais une erreur. Les
valeurs chargées deviennent la base des options du relais qui
déclarent une variable d'environnement (`env = "…"` dans
`cli.rs`).

Précédence (une option, trois sources) :

```text
explicit CLI flag  >  .env variable  >  built-in default
```

- un drapeau passé sur la ligne de commande l'emporte TOUJOURS ;
- sinon la variable (qu'elle soit chargée depuis `.env` ou
  définie dans le vrai environnement du processus — même
  variable, même effet) ;
- sinon le défaut intégré de clap.

`.env` ne tient qu'UNE valeur par variable : dotenvy applique la
PREMIÈRE occurrence d'une variable (les doublons sont silencieusement
ignorés) et n'accumule jamais ; le vrai environnement du processus
l'emporte aussi sur le fichier. Plusieurs seeds de bootstrap ou
plusieurs upstreams DNS exigent donc des drapeaux `--bootstrap` /
`--dns-upstream` répétés sur la ligne de commande.

Le tableau complet des variables `SCONE_*` (réseau, écoute P2P,
bootstrap, genèse, UPnP, surface DNS, métriques, profil de
topologie, ancre) et la sémantique de chacune vit dans
[`relay.md`](relay.md#configuration-par-environnement-env) ; le
modèle commenté est `.env.example` à la racine du dépôt (copier
vers `.env`, ne jamais le committer — `.env` est gitignored).

## Sémantique des niveaux (convention du code)

- `error!` : échecs locaux fatals (signer désactivé sur un état de
  signataire illisible/corrompu, store inutilisable, échec de
  récupération de fork profond, passphrase d'ancre non persistée) —
  les échecs réseau sont engloutis en `warn!` et seuls les échecs
  locaux fatals remontent comme erreur typée depuis `run()`) ;
- `warn!` : entrées réseau hostiles ou malveillantes repoussées,
  transactions périmées évincées du mempool, transactions
  conflictuelles abandonnées pendant la production, échecs
  sortants/entrants, discordance de clé DHT ;
- `info!` : bloc accepté/produit, écoute p2p, pair connecté,
  démarrage du relais (points de contrôle de l'état) et la ligne
  de flavor de build au démarrage —
  `build: mainnet-enabled` / `build: standard` ;
- `debug!` : détails internes (transaction acceptée, pair
  déconnecté, aller-retour RPC, listing du coffre de clés,
  événements kad sous `SCONE_KAD_DEBUG`) ;
- `trace!` : réservé aux payloads (aucun site d'appel
  aujourd'hui).

Les crates de bibliothèque (`scone-network`, …) émettent des
macros `tracing` mais **n'initialisent aucun subscriber** :
l'initialisation vit uniquement dans le binaire `scone` (avant
la répartition des sous-commandes — le relais a besoin de ses
journaux dès le démarrage).


## Usage simple (cli-simplify-v1)

- **Pas besoin de `--peer`** : les commandes parlent au relais
  local par défaut ; la variable d'env `SCONE_PEER` ou `--peer`
  surcharge.
- **Pas besoin de `--identity`** quand le coffre de clés ne
  contient qu'une seule identité (elle est utilisée
  silencieusement ; plusieurs → l'erreur les liste).
- **STRICT : une identité EST requise pour `domain register`** —
  un coffre vide échoue avec une erreur typée pointant
  `scone identity generate` (une clé fabriquée à la volée
  orphelinerait le domaine à la première opération d'admin — la
  seed affichée une fois n'est pas un coffre). Pour enregistrer
  POUR une autre clé sans aucun coffre : `--to <pubkey>`
  (pow-to-owner — pas de signature, la preuve est toute
  l'autorisation). (strict-cli-lifecycle)
- Supprimés : `show`, `submit`, `lookup`, `owned` — pas de stubs ;
  clap rejette ces noms avec son erreur générique « unrecognized
  subcommand » plus une suggestion du nom le plus proche.
- Nouveau : `scone domains`, `scone domain get|set`
  (enregistrement unique, pas de fichier). Le renouvellement est
  FUSIONNÉ dans `domain register` (unify-lifecycle-commands) — le
  verbe `domain renew` n'existe plus.

```bash
scone identity generate me
scone domain register example.scone   # claim OU renewal (unifié)
scone domain set example.scone A 192.0.2.1
```

`domain register` sur un nom QUI VOUS APPARTIENT exécute le
renouvellement : **terme fixe d'un an** à partir du plus tardif
entre l'échéance courante et maintenant — plafonné au maximum
anti-hoarding (`now + 3 ans`) pour un domaine empilé près du
plafond. Les durées de renouvellement sont fixes : aucun choix.
L'extension est sans perte (renouveler tôt ne perd rien) — pas
de gate : à vous = renouvellement exécuté et annoncé.

## Cycle de vie des TLD (cli-unified-lifecycle)

`scone tld register <tld>` est le point d'entrée TLD en une
seule commande, symétrique de `scone domain register` — l'état
on-chain est lu D'ABORD, puis la bonne action s'exécute :

- **TLD libre** → enregistré, né **OPEN par défaut** (pas de
  drapeau : espace de noms en libre-service, 1/3 du PoW).
  `--private` opte pour un espace de noms possédé au prix
  complet du PoW, avec une confirmation explicite (décision
  économique). `--yes` saute les confirmations (scripts).
- **à vous, expirant sous 90 jours** → le RENOUVELLEMENT est
  proposé (PoW lié à l'expiration courante — anti-réutilisation)
  et s'exécute après confirmation.
- **à vous, loin de l'expiration** → erreur typée (exit non
  nul) : le terme TLD est FIXE (`now + 3 ans`) — renouveler tôt
  PERDRAIT le temps restant, donc le renouvellement anticipé
  n'est pas offert (le gate protège, il ne complique pas).
  L'erreur porte le hint « renewal opens within 90 days of
  expiry » (strict-cli-lifecycle : un exit 0 silencieux ferait
  croire à un script qu'il a renouvelé).
- **à quelqu'un d'autre** → erreur typée propre avec le
  propriétaire.

```bash
scone tld register mytld            # open birth (default)
scone tld register mytld --private  # owned namespace (confirm)
```

De même, `scone domain register <nom>` sur un nom que vous
possédez exécute le renouvellement (terme fixe d'un an) au lieu
d'échouer.

## Délégués de records (KSK/ZSK-lite)

Le propriétaire d'un domaine peut accorder à D'AUTRES clés le droit
de signer `domain update` en son nom — le jeu de délégués de records
(au plus 3 clés, remplacement intégral à chaque `set`) :

```bash
# octroi (signé par le propriétaire ; le CLI récupère la génération
# CAS du namespace et lie la transaction à elle) :
scone domain delegates set example.uip \
  --keys <pk-hex-1>,<pk-hex-2> --identity alice

# consultation (aucune identité requise) :
scone domain delegates list example.uip

# tout révoquer :
scone domain delegates clear example.uip --identity alice
```

Un délégué signe ensuite `domain update` exactement comme le
ferait le propriétaire (`--identity deleg`) — on-chain le champ
`owner` de la tx est le propriétaire du DOMAINE tandis que la
signature vient du délégué. Le don est exclusif au propriétaire (un
délégué ne peut pas gérer le jeu), vidé par un transfert, et chaque
`set`/`clear` incrémente le compteur de mutations du namespace (CAS
anti-replay — voir `transactions.md`) : un `set` rejoué ne peut pas
ressusciter des clés révoquées. L'update d'un délégué révoqué est
refusée localement (« neither the owner nor a record delegate »).

## Commandes avancées

Les défauts ci-dessus couvrent l'usage quotidien. Pour les
scripts et l'interopérabilité web/API, la surface complète reste
disponible — chaque action n'a qu'UN seul chemin canonique ; les
anciens alias racine (`show`, `lookup`, `owned`, `renew`) ont été
supprimés (remove-cli-aliases) ; `domain
info` subsiste comme alias de compatibilité de `domain get`.

La surface hexadécimale hors ligne (sous-commandes `tx
build|sign|verify`, `submit tx`) a été SUPPRIMÉE
(remove-tx-surface-v2) : les opérations M8b qu'elle servait sont
désormais des commandes de premier plan — `scone tld open <tld>`
(ouverture en libre-service à sens unique, PoW d'ouverture miné
pour vous), `scone tld transfer <tld> <owner>`, `scone tld
revoke <tld>`, `scone domain assign <nom> <owner>`
(propriétaire du TLD, TLD privé) et `scone domain transfer
<nom> <owner>`. Chacune : signature,
soumission, txid. Les chemins de transfert (`tld transfer`,
`domain transfer`) pré-lisent l'état on-chain côté client — la
preuve de cession est liée à la génération courante — tandis que
`tld open`, `tld revoke` et `domain assign` ne font aucune
lecture d'état côté client (le nœud fait respecter). La
résolution d'enregistrement DNS a déménagé
vers `scone domain get <nom>` (ex-`record get`, ex-`domain resolve`) ; le canal
de publication hexadécimal (`record put`) a disparu — l'édition
passe par `domain set` / `domain update`.
- `scone domain mine <nom>` — mine UNIQUEMENT la preuve
  d'enregistrement (aucune identité, aucun coffre, aucune
  soumission) : imprime le JSON `scone-domain-pow-v1` `{type,
  name, network_id, epoch, proof}` sur stdout. Minage délégué :
  une autre machine mine pour vous, vos clés ne quittent jamais
  la vôtre. Pipez ou redirigez : `scone domain mine n.tld |
  scone domain register n.tld --proof -` (ou `--proof FICHIER`).
  La preuve est liée au nom et au réseau ; la fraîcheur est
  vérifiée par le nœud à l'application (fenêtre de validité
  d'époque). Avec `--mine-for <pubkey>`, il mine à la place une
  preuve POW-TO-OWNER pour ce bénéficiaire et imprime le JSON
  `scone-domain-pow-to-v1` `{type, name, network_id, epoch,
  pow_to, proof}` — consommable UNIQUEMENT par `register --to` /
  `renew --to` pour exactement cette clé (le challenge l'embarque ;
  contrôle insensible à la casse à la soumission).
- Soumission pow-to-owner (aucun coffre nécessaire — `--to` est
  exclusif de `--identity`) : `scone domain register <nom> --to
  <pubkey>` et `scone tld register <tld> --to <pubkey>`
  soumettent une preuve dont le challenge lie la clé du
  BÉNÉFICIAIRE (sur un nom à vous, la même commande renouvelle —
  pow-to-owner, preuve liée à l'expiry courante) ; l'`owner` en
  chaîne est recalculé
  comme `hash(pubkey_dest)` (un renew ne change JAMAIS de mains).
  Les commandes `--to` minent toujours leur preuve sur place —
  `--proof` n'existe QUE sur `domain register --to` : avec
  `--proof -`/FICHIER elle consomme un payload
  `scone-domain-pow-to-v1` et refuse une preuve minée pour UN
  AUTRE bénéficiaire. Style pipe : `scone domain mine n.tld
  --mine-for <pubkey> | scone domain register n.tld --to
  <pubkey> --proof -`.
- `scone domain export <nom> [--out FILE]` — écrit le jeu
  d'enregistrements vérifié sur chaîne dans le format exact de
  `update --file` (aller-retour garanti : export → édition →
  update re-soumet le même jeu). Sans `--out` : stdout.
  `domain get` utilise le même format lisible pour la partie
  record, donc `domain get <nom> > zone.txt` est directement
  importable (`--raw` ajoute les encodages hex). Les types
  sans syntaxe textuelle (GEOROUTE, inconnus) sont exportés
  comme des avertissements en commentaire — jamais abandonnés
  silencieusement.

`domain get` affiche aussi le `domain_id` et le `valid_until`.

## Benchmark PoW (`scone bench-pow`)

Mesure le taux Argon2id de la machine (le même hash que la
vérification consensus) et traduit les difficultés du réseau en
temps d'attente concrets :

```
scone bench-pow --network mainnet [--seconds 5] [--cores N]
```

Sortie : H/s total et par worker, puis le temps moyen attendu par
opération — transfert/assignation (taux plat), ouverture d'un TLD
privé, et planchers d'enregistrement TLD/domaine (courbe de
longueur). Le réseau choisi détermine les paramètres Argon2id
(mainnet : 64 MiB, t=2 — le tarif réel ; testnet/devnet : profil
léger, preuves en millisecondes). Chaque worker alloue son propre
lane Argon2id — 64 MiB sur le profil mainnet, 8 KiB sur le
profil léger testnet/devnet — dimensionnez `--cores` sur la RAM
disponible.

## Builds : dev vs mainnet-activé (mainnet-compile-gate)

Le journal de démarrage du relais s'ouvre sur la flavor de build
(`build: mainnet-enabled` / `build: standard`) — l'équivalent
exécutable du marqueur `+mainnet` de `--version`.

Les builds par défaut n'embarquent **pas de mainnet** : `scone
relay --network mainnet` (et les services du registrar avec
`SCONE_NETWORK=mainnet`) échouent immédiatement avec une
indication de rebuild. Le binaire officiel distribué est
construit avec :

```bash
cargo build --release --features mainnet
```

`scone --version` affiche `+mainnet` sur de tels builds.

Le `--network` par défaut suit aussi le build : `mainnet` sur
les builds mainnet-activés (le binaire officiel dépose les
utilisateurs sur le réseau de production), `testnet` sinon. Un
drapeau `--network` explicite ou la variable d'env
`SCONE_NETWORK` l'emportent toujours (ordre clap : drapeau >
env > défaut). C'est un garde-fou opérationnel (exécutions
accidentelles, binaires dev/CI) — pas une frontière
cryptographique ; les règles de consensus restent la défense
contre les nœuds hostiles.

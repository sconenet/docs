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
  des journaux.

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
un démon — il doit journaliser son activité (écoute RPC/p2p,
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

Les variables répétables (`SCONE_BOOTSTRAP`, `SCONE_DNS_UPSTREAM`)
s'accumulent : une ligne par valeur dans `.env` (dotenvy ajoute).

Le tableau complet des variables `SCONE_*` (réseau, écoute P2P,
bootstrap, genèse, mDNS, UPnP, surface DNS, métriques, profil de
topologie, ancre) et la sémantique de chacune vit dans
[`relay.md`](relay.md#configuration-par-environnement-env) ; le
modèle commenté est `.env.example` à la racine du dépôt (copier
vers `.env`, ne jamais le committer — `.env` est gitignored).

## Sémantique des niveaux (convention du code)

- `error!` : échecs (aucun site d'appel aujourd'hui — les échecs
  réseau sont engloutis en `warn!` et seuls les échecs locaux
  fatals remontent comme erreur typée depuis `run()`) ;
- `warn!` : entrées réseau hostiles ou malveillantes repoussées,
  transactions périmées évincées du mempool, transactions
  conflictuelles abandonnées pendant la production, échecs
  sortants/entrants, discordance de clé DHT ;
- `info!` : bloc accepté/produit, écoute RPC/p2p, pair connecté,
  démarrage du relais (points de contrôle de l'état) ;
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
- Supprimés (les stubs suggèrent le remplacement) : `show`,
  `submit`, `lookup`, `owned`.
- Nouveau : `scone domain renew <nom>`, `scone domains`,
  `scone domain get|set` (enregistrement unique, pas de
  fichier).

```bash
scone identity generate me
scone domain register example.scone
scone domain set example.scone A 192.0.2.1
scone domain renew example.scone
```

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
- **à vous, loin de l'expiration** → rien à faire
  (`tld renew --force` pour renouveler quand même).
- **à quelqu'un d'autre** → erreur typée propre avec le
  propriétaire.

```bash
scone tld register mytld            # open birth (default)
scone tld register mytld --private  # owned namespace (confirm)
scone tld renew mytld               # explicit renewal
```

De même, `scone domain register <nom>` sur un nom que vous
possédez et qui expire sous 90 jours propose le renouvellement
au lieu d'échouer.

## Commandes avancées

Les défauts ci-dessus couvrent l'usage quotidien. Pour les
scripts et l'interopérabilité web/API, la surface complète reste
disponible — chaque action n'a qu'UN seul chemin canonique ; les
anciens alias racine (`show`, `lookup`, `owned`, `renew`) ont été
supprimés (remove-cli-aliases), tout comme la mention `domain
info`, qui ne correspondait à aucune commande réelle.

La surface hexadécimale hors ligne (sous-commandes `tx
build|sign|verify`, `submit tx`) a été SUPPRIMÉE
(remove-tx-surface-v2) : les opérations M8b qu'elle servait sont
désormais des commandes de premier plan — `scone tld open <tld>`
(ouverture en libre-service à sens unique, PoW d'ouverture miné
pour vous), `scone tld transfer <tld> <owner>`, `scone tld
revoke <tld>`, `scone domain assign <nom> <owner>`
(propriétaire du TLD, TLD privé) et `scone domain transfer
<nom> <owner>`. Chacune : lecture de l'état, signature,
soumission, txid. La résolution d'enregistrement DNS a déménagé
vers `scone domain resolve <nom>` (ex-`record get`) ; le canal
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
  d'époque).
- `scone domain export <nom> [--out FILE]` — écrit le jeu
  d'enregistrements vérifié sur chaîne dans le format exact de
  `update --file` (aller-retour garanti : export → édition →
  update re-soumet le même jeu). Sans `--out` : stdout.
  `domain resolve` utilise le même format lisible, donc
  `resolve > zone.txt` est directement importable. Les types
  sans syntaxe textuelle (GEOROUTE, inconnus) sont exportés
  comme des avertissements en commentaire — jamais abandonnés
  silencieusement.

`domain get` affiche aussi le `domain_id` et le `valid_until`.

## Builds : dev vs mainnet-activé (mainnet-compile-gate)

Les builds par défaut n'embarquent **pas de mainnet** : `scone
relay --network mainnet` (et les services du registrar avec
`SCONE_NETWORK=mainnet`) échouent immédiatement avec une
indication de rebuild. Le binaire officiel distribué est
construit avec :

```bash
cargo build --release --features mainnet
```

`scone --version` affiche `+mainnet` sur de tels builds, et le
journal de démarrage du relais indique
`build: mainnet-enabled`.

Le `--network` par défaut suit aussi le build : `mainnet` sur
les builds mainnet-activés (le binaire officiel dépose les
utilisateurs sur le réseau de production), `testnet` sinon. Un
drapeau `--network` explicite ou la variable d'env
`SCONE_NETWORK` l'emportent toujours (ordre clap : drapeau >
env > défaut). C'est un garde-fou opérationnel (exécutions
accidentelles, binaires dev/CI) — pas une frontière
cryptographique ; les règles de consensus restent la défense
contre les nœuds hostiles.

# CLI (`scone`) — logging et verbosité

> Comportement normatif du logging du binaire `scone`. Toute divergence
> doc/code = bug (corriger la source de vérité).

## Principes

- **stdout est réservé aux sorties de commandes** (`scone show`,
  `identity list`, `status`, …). Un script peut rediriger stdout sans
  jamais y trouver de logs — le piping reste fiable. Exception
  assumée : `scone relay` émet des lignes de données
  `p2p: <multiaddr>`, `dns: <addr>` et `metrics: <addr>` sur
  stdout dès que les écouteurs sont liés (données
  exploitables pour `--bootstrap` / configuration, pas des logs ;
  émises une seule fois, flushées immédiatement).
- **Tous les logs vont sur stderr** (format compact `tracing`), y
  compris ceux du relay — le relay loggait déjà sur stderr avant la
  migration.
- Les messages d'erreur utilisateur du CLI (`scone: <erreur>`) restent
  sur stderr : c'est de l'UX, pas des logs.

## Niveaux et `--verbose` / `-v`

| Invocation | Niveau effectif |
|---|---|
| `scone <cmd>` (sans `-v`) | `warn` |
| `scone relay` (sans `-v`) | `info` |
| `scone -v <cmd>` / `scone relay -v` | `info` |
| `scone -vv …` | `debug` |
| `scone -vvv …` (et plus) | `trace` |

`-v`/`--verbose` est un drapeau **global et répétable** : il s'accepte
avant ou après la sous-commande (`scone -vv identity list` et
`scone identity list -vv` sont équivalents).

**Pourquoi `scone relay` reste à INFO par défaut** : le relay est un
daemon — il doit logger son activité (écoute RPC/p2p, blocs acceptés
et produits, pairs connectés, rejets) pour être opérable, sans exiger
un drapeau. Les commandes one-shot, elles, restent silencieuses (WARN)
pour ne pas polluer un terminal ou un script.

## `RUST_LOG` (override)

Si la variable d'environnement `RUST_LOG` est définie, elle **remplace
entièrement** le niveau dérivé de `-v` (syntaxe `env-filter`) :

```bash
RUST_LOG=debug scone identity list        # debug sans -v
RUST_LOG=error scone -vv relay            # error gagne sur -vv
RUST_LOG=scone_network=trace,info scone relay   # par-cible
```

## Configuration par environnement (`.env`)

Avant le parsing des arguments, `scone` charge un fichier `.env`
depuis le répertoire courant (`dotenvy::dotenv()` dans `main.rs`).
Fichier absent = silence (no-op), jamais une erreur. Les valeurs
chargées deviennent la ligne de base des options relay qui
déclarent une variable d'environnement (`env = "…"` dans `cli.rs`).

Priorité (une option, trois sources) :

```text
flag CLI explicite  >  variable de .env  >  défaut intégré
```

- un flag passé sur la ligne de commande gagne TOUJOURS ;
- sinon la variable (que ce soit `.env` chargé ou l'environnement
  réel du processus — même variable, même effet) ;
- sinon le défaut intégré de clap.

Les variables répétables (`SCONE_BOOTSTRAP`, `SCONE_DNS_UPSTREAM`)
accumulent : une ligne par valeur dans `.env` (dotenvy append).

Le tableau complet des variables `SCONE_*` (réseau, écoute P2P,
bootstrap, genèse, mDNS, UPnP, surface DNS, métriques, anchor) et
la sémantique de chacune vit dans
[`relay.md`](relay.md#configuration-par-environnement-env) ; le
modèle commenté est `.env.example` à la racine du dépôt (copier
vers `.env`, jamais commiter — `.env` est gitignoré).

## Sémantique des niveaux (convention du code)

- `error!` : échecs (aucun site aujourd'hui — les échecs réseau sont
  avalés comme `warn!` et seuls les échecs locaux fatals remontent en
  erreur typée du `run()`) ;
- `warn!` : entrées réseau hostiles ou malveillantes écartées, tx
  périmées éjectées du mempool, tx conflictuelles abandonnées en
  production, échecs outbound/inbound, key mismatch DHT ;
- `info!` : bloc accepté/produit, écoute RPC/p2p, pair connecté,
  démarrage du relay (checkpoints d'état) ;
- `debug!` : détails internes (tx acceptée, pair déconnecté, round
  trip RPC, listing keystore, événements kad sous `SCONE_KAD_DEBUG`) ;
- `trace!` : réservé aux payloads (aucun site aujourd'hui).

Les crates lib (`scone-network`, …) émettent des macros `tracing`
mais **n'initialisent aucun subscriber** : l'initialisation vit
uniquement dans le binaire `scone` (avant le dispatch des
sous-commandes — le relay a besoin de ses logs dès le boot).

# Serveur DNS UDP + TCP (M6 + M7b) — `scone dns`

> Spécification normative de la surface DNS de Scone.
> Toute divergence doc/code = bug (corriger la source de vérité).

## Vue d'ensemble

Le **serveur DNS** est une surface optionnelle du relay : il répond
aux requêtes DNS **UDP et TCP** (RFC 7766, même port) pour les noms
Scone **uniquement à partir de données vérifiées contre la chaîne**,
et propose un **fallback récursif optionnel** (`addr:port`) pour le
reste. Il ne possède ni chaîne, ni store, ni DHT : la résolution
passe par le relay (processus), via le RPC de contrôle local — le
même chemin vérifié que le CLI.

```text
              UDP+TCP :5353 (par défaut 127.0.0.1)
                    │
             scone relay --dns 127.0.0.1:5353
                    │  ┌────────────────────────────────┐
                    ├──┤ Cache borné (qname,qtype)      │
                    │  └────────────────────────────────┘
                    │  nom de forme Scone ?
                    │  ├─ OUI → recherche d'apex par suffixe
                    │  │        → RPC `resolve_local` (relay)
                    │  │           chaîne + record vérifié (hash,
                    │  │           owner, séquence) → réponses
                    │  │        apex inconnu sur toute la chaîne → NXDOMAIN
                    │  │        apex sans record publié → NODATA
                    │  └─ NON → fallback amont (si configuré)
                    │           sinon REFUSED
                    │
                    │  budget de réponse :
                    │    UDP → 512 o (pas d'EDNS) ; dépassement =
                    │          coupe au dernier record + TC=1 → le
                    │          client rejoue en TCP
                    │    TCP → 64 KiB (préfixe longueur 2 o,
                    │          RFC 1035 §4.2.2) : set complet
```

## Démarrage

```bash
scone relay --dns 127.0.0.1:5353            # surface DNS seule
scone relay --dns 0.0.0.0:53 \
            --dns-upstream 1.1.1.1:53 \
            --dns-upstream 8.8.8.8:53       # + fallback récursif
```

Le relay annonce l'adresse effective sur stdout (contrat
machine-lisible, comme `p2p:`) : une ligne `dns: <addr>` dès le bind.

Client de validation : `scone dig <name> --dns 127.0.0.1:5353
--qtype A` (A, AAAA, TXT, MX, NS, CNAME, ANY).

## Chemin de résolution (AUTORITÉ = la chaîne, toujours)

1. **Cache** : clé `(qname, qtype)`, capacité 100 000 entrées par
   défaut (`--dns-cache-limit` pour outrepasser), TTL 60 s
   (autoritaire) / clampé à 600 s (fallback). Le cache négatif
   (NXDOMAIN/NODATA) suit les mêmes règles ; chaque entrée porte son
   origine (autoritaire ou fallback) pour que le bit AA reste honnête
   sur un hit. **Jamais de cache pour SERVFAIL** (défaillance locale
   transitoire) **ni REFUSED** (état de configuration) : recalculés à
   chaque requête. Éviction FIFO à la capacité.
2. **Forme Scone ?** Pré-filtre strict : labels `[a-z0-9-_]` (1..63),
   ≥ 2 labels, TLD 1..63 octets `[a-z0-9-]` (cf. `TldName::MAX_LEN`),
   longueur totale ≤ 253 — **et TLD hors de la racine ICANN**
   (`is_icann_tld`, liste figée IANA + RFC 2606 : `com`, `fr`, `org`…
   n'appartiennent jamais à Scone).
   Un nom qui ne peut PAS être un nom Scone (ex. `www.foo_bar`,
   underscore dans le TLD) part au fallback ; un nom Scone-possible
   (`absent.uip`) relève de la chaîne : non enregistré → NXDOMAIN
   autoritaire, même sans upstream. Un TLD ICANN (`example.com`) est
   forwardé vers les upstreams si configurés, REFUSED sinon — jamais
   résolu autoritairement par Scone.
3. **Recherche d'apex** : suffixes du plus long au plus court (min.
   2 labels). Premier suffixe dont le relay connaît l'état on-chain
   = apex ; le record set sert le qname entier (sous-noms inclus —
   comportement wildcard implicite, décision ouverte).
4. **`resolve_local` (RPC interne)** : un seul aller-retour, jamais
   de waiter DHT. État on-chain (`registered`, owner, séquence) +
   record du cache DHT local **revérifié** (owner, séquence,
   record_hash == engagement on-chain). Un record non conforme ne
   répond jamais — disponibilité ≠ autorité.
   - non enregistré → NXDOMAIN ;
   - enregistré sans record valide → NODATA (NOERROR, 0 réponses) ;
   - record vérifié → réponses filtrées par qtype.

## Codec (RFC 1035, sous-ensemble strict)

- Requête : 12 octets d'en-tête, **une** question, QCLASS = IN,
  **pas de compression** (pointeur = rejet). Labels lowercasés et
  validés. QR=1, QDCOUNT≠1, labels invalides → **silence** (le
  garbage ne coûte rien).
- Réponse : QR|RD(écho)|RA, **AA uniquement sur les réponses
  autoritaires** (chaîne, fraîches ou cachées — jamais sur le
  fallback ni les RCODEs d'erreur), question recopiée telle quelle,
  réponses **non compressées** (owner = qname).
- Types servis : A, AAAA, CNAME, NS, MX, TXT, **PTR, SRV, CAA**
  (+ Unknown re-encodé tel quel ; 9 types natifs au total, rdata
  encodés par RFC — 1035/2782/8659 ; CAA strict : bit critique 0x80
  seul, tags minuscules). Une requête A/AAAA inclut les CNAME du set
  (suivi
  d'alias). Requête ANY (255) : tout le set filtré par le codec.
- Limites (M7b) : **le budget de réponse dépend du transport** —
  512 octets en UDP (charge utile classique RFC 1035 §2.3.4, pas
  d'EDNS), 65 535 en TCP. Un set dépassant le budget est coupé au
  dernier record qui tient avec **TC=1** (troncature honnête, RFC
  1035 §4.2.1) : le client rejoue en TCP (RFC 7766), où le set
  complet est servi. Une réplique amont plus grande que le budget
  UDP est ré-encodée coupée (TC=1) au lieu d'être livrée telle
  quelle ; sur TCP elle passe byte pour byte. La troncature
  s'applique aussi aux hits de cache (le budget est appliqué au
  moment de la réponse, pas du remplissage du cache).
- RCODEs : 0 NOERROR (avec ou sans réponses), 2 SERVFAIL (failure
  RPC locale), 3 NXDOMAIN (autoritaire), 5 REFUSED (hors Scone sans
  upstream).

## Fallback récursif

- Amonts `addr:port` (UDP), essayés dans l'ordre, timeout 2 s par
  amont, socket **connectée** à l'amont (filtrage noyau : un paquet
  d'une autre source n'est jamais accepté). La réponse amont n'est
  acceptée que si elle **matche la requête** : même TXID, QR=1,
  question (QNAME+QTYPE+QCLASS) recopiée octet pour octet. Une
  réponse non conforme → SERVFAIL, et rien n'est mis en cache.
- La réponse amont validée est renvoyée telle quelle ; ses réponses
  sont mises en cache avec l'origine `fallback` (AA jamais positionné,
  même sur un hit de cache) et le TTL clampé à 600 s. Les RCODEs
  d'erreur de l'amont (hors NOERROR/NXDOMAIN) ne sont pas cachés.
- **Privé par défaut** : pas d'amont = REFUSED sur les noms hors
  forme Scone.

## TCP (M7b, RFC 7766)

Même port que l'UDP (second bind TCP sur l'adresse `--dns`). Le
transport suit RFC 1035 §4.2.2 : chaque message est préfixé de sa
longueur sur **2 octets** big-endian.

- **Requête** : longueur préfixe ≥ 1 et ≤ 512 octets (une requête
  DNS tient la charge utile classique — RFC 7766 §6 ; pas d'EDNS
  servi). Hors bornes → fermeture immédiate de la connexion.
- **Réponse** : budget 65 535 octets (le maximum du préfixe 2 o) —
  un set trop grand pour l'UDP y est servi **en entier**, TC=0.
- **Pipelining** : plusieurs requêtes par connexion sont servies
  en série, dans l'ordre (une écriture = une réponse préfixée).
- **Timeout de lecture** : 5 s par lecture (préfixe ou corps) — un
  client muet est déconnecté, son slot libéré.
- **Connexions concurrentes** : 64 max (sémaphore) ; la 65e est
  rejetée immédiatement (connexion fermée), jamais mise en file —
  aucune croissance non bornée de tâches ou de mémoire.
- Erreurs lecture/écriture/timeout : log debug + fermeture propre.
  **Jamais de panic sur entrée réseau.**
- Le garbage (requête incompréhensible) reçoit le silence mais ne
  ferme pas la connexion : la requête suivante (valide) est servie.

```text
client                     serveur
  │ [len u16][requête]        │
  │ ─────────────────────►   │ cache → autoritaire → fallback
  │ ◄─────────────────────   │ [len u16][réponse ≤ 64 KiB]
  │ (UDP avait répondu TC=1 → le client rejoue ici)
```

## Bornes et garde-fous

| Garde-fou | Valeur |
|---|---|
| Datagramme UDP (req/rép) | 4096 octets (rejet au-delà) |
| Budget réponse UDP (TC=1 au-delà) | 512 octets (pas d'EDNS) |
| Requête TCP max | 512 octets (fermeture au-delà) |
| Réponse TCP max | 65 535 octets |
| Connexions TCP concurrentes | 64 (sémaphore, rejet immédiat) |
| Timeout lecture TCP | 5 s |
| Cache | 1024 entrées (FIFO) |
| TTL autoritaire | **par enregistrement** (`record.ttl`, bornes 1..=2 592 000 s ; le 60 s constant historique ne reste que pour les réponses de service) |
| TTL fallback (clamp) | 600 s |
| Requêtes UDP en parallèle | 256 (sémaphore ; au-delà : drop, le client retente) |
| Timeout par amont | 2 s |

Règles projet respectées : jamais de panic sur données réseau ;
tout décodage strict ; le serveur ne touche ni chaîne ni store ni
DHT (le relay reste l'unique autorité de vérification).

## Tests

- Unitaires (`scone-network::dns`) : codec strict (rejets QR,
  compression, labels), réponses autoritaires, NODATA, NXDOMAIN,
  REFUSED sans amont, fallback vers un mock UDP, cache borné,
  chunking TXT, parsing des amonts ; M6 fix-up : honnêteté du bit AA
  (autoritaire vs fallback vs erreurs, y compris hit de cache),
  troncature TC=1 d'un set trop grand, rejet d'une réponse amont
  (TXID ou question ne matchant pas), non-cache de SERVFAIL/REFUSED.
- M7b : budget par transport (UDP 512 o → TC=1, TCP → set complet),
  troncature d'un hit de cache, `run_tcp` réel (set > 512 servi
  entier, pipelining de 2 requêtes, requête oversize → fermeture,
  client muet → timeout 5 s, garbage puis requête valide sur la
  même connexion), réplique amont > 512 ré-encodée TC=1 en UDP /
  byte pour byte en TCP.
- Intégration réelle (`tests/dns_server.rs`) : un relay complet
  (production devnet incluse) + enregistrement + UpdateDomain + PutRecord,
  puis requêtes UDP réelles : A vérifié (bit AA), TXT, NODATA AAAA,
  NXDOMAIN, fallback via mock, re-quête, sous-nom via apex.
- E2E binaire (`crates/scone/tests/dns_cli.rs`) : le binaire
  `scone` réel — `relay --dns`, `identity generate`, `domain
  register`, `domain update --file`, puis `scone dig` (A vérifié,
  TXT, NXDOMAIN, REFUSED) sur un vrai port UDP.

## Décisions ouvertes

- Pas de EDNS (le budget UDP reste 512 o ; la troncature TC=1 +
  TCP couvre le besoin devnet).
- Le record set ne porte pas de nom par record : tout le set sert
  tout sous-nom de l'apex (wildcard implicite). Un adressage par
  enregistrement exigerait d'étendre `DnsRecord` (protocol).
- Le fallback transfère la requête brute : un amont malicieux peut
  répondre n'importe quoi (réponse non validée — c'est le modèle
  récursif standard ; les noms Scone ne passent JAMAIS par là).

## GeoDNS (routage géographique, opt-in)

> Spécification normative. Toute divergence doc/code = bug.

### Vue d'ensemble

Un domaine peut publier des règles de **routage géographique** dans
son record set signé : des records `GEOROUTE` (type privé Scone
`32 768`, hors plage IANA). Un resolver dont le GeoDNS est armé
(`--geo-dns`) répond aux questions A/AAAA de ces domaines avec
**l'adresse du endpoint choisi par la règle**, au lieu d'un record
A/AAAA statique. Tout le reste (domaines sans GEOROUTE, autres
qtypes, resolvers sans `--geo-dns`) suit le chemin DNS classique,
**octet pour octet** — GeoDNS est 100 % opt-in.

### Record GEOROUTE (wire)

Un GEOROUTE = un **scope** + ≤ 32 **endpoints** (tri canonique au
décodage : priorité croissante, poids décroissant, cible croissante ;
doublon = rejet) :

```text
GeoScope   = 0x00 (monde)
           | 0x01 continent.v (code UN M49 : AF=2, AS=142, EU=150,
                                NA=3, SA=5, OC=9, AN=999)
           | 0x02 country.v   (code ISO 3166-1 numérique : US=840,
                                TW=158, FR=250…)
GeoTarget  = 0x00 octets[4]  (adresse IPv4 binaire)
           | 0x01 octets[16] (adresse IPv6 binaire)
GeoEndpoint = target || priority.v || weight.v
GEOROUTE    = scope || count.v(≤32) || endpoints…
```

Bornes de validation (décodage) : `priority > 0`, `weight ≤ 100`
(`weight = 0` = endpoint inactif, jamais élu), TTL dans
1..=2 592 000 s comme tout record. Un pays/continent inconnu des
tables locales est **autorisé** au wire : le routage retombe
simplement sur un scope supérieur.

### Sélection (résolution)

1. **Scope** — spécificité décroissante : **pays exact → continent →
   monde**. Le continent est dérivé du pays par la table locale
   (`continent_of_country`) si le resolver ne connaît que le pays ;
   un pays inconnu de la table retombe sur le continent explicite ou
   le monde. Aucune route qui matche (y compris sans scope World) →
   comportement DNS classique, comme si le GEOROUTE n'existait pas.
2. **Priorité** — sémantique SRV-like : seul le groupe de
   **priorité minimale** est éligible ; les priorités supérieures
   sont des niveaux de **failover**.
3. **Poids** — dans le groupe élu, tirage **pondéré déterministe** :
   hash stable (SplitMix64) d'un nonce (compteur de requêtes du
   resolver) contre les poids cumulés. Pas de RNG : deux requêtes de
   même nonce donnent le même résultat, des nonces croissants
   répartissent proportionnellement aux poids. `weight = 0` =
   inactif, jamais élu (même seul — on passe au niveau suivant).
4. **Failover santé** — un endpoint **Unhealthy** est écarté ; si
   tout le niveau minimal est indisponible, on passe au niveau de
   priorité suivant. **Unknown compte comme disponible** (on ne
   pénalise pas l'inconnu). Tout indisponible → comportement DNS
   classique.

### Réponse

- Une seule adresse : **A si IPv4, AAAA si IPv6** — filtrage strict
  par famille selon le qtype (une question A ne répond JAMAIS une
  IPv6, et réciproquement).
- TTL = celui du record GEOROUTE d'origine (borné au décodage par
  les règles TTL générales).
- AA positionné (réponse autoritaire — le GEOROUTE est un record
  signé vérifié comme les autres).
- Réponse vide après filtrage (aucun candidat de la bonne famille,
  tout Unhealthy, pas de route matchante) → le resolver poursuit sur
  son chemin classique (typiquement NODATA : NOERROR, 0 réponses) ;
  les rcodes d'erreur suivent les règles générales (jamais cachés).
- Un record A/AAAA classique coexistant dans le set PRIME sur le
  geo (le geo ne s'applique qu'en l'absence de réponse wire pour le
  qtype).

### Cache : décision explicite

La décision de routage geo n'est **JAMAIS cachée** sous
`(qname, qtype)` : elle dépend du scope du client du resolver, et
une telle clé servirait la route US à un client TW. Les réponses
geo sont recalculées à chaque requête (coût : un select_scope +
tirage pondéré, µs) et contournent le cache. Le cache classique
(qname, qtype) reste inchangé pour tout le reste — y compris les
NODATA produits quand le geo ne décide rien. Les rcodes d'erreur
restent jamais cachés.

### Health checks (`--health-check`)

- `HealthCache` par cible (`GeoTarget`) : `probe` = lecture non
  bloquante (statut frais, ou Unknown), **jamais** de check déclenché
  depuis le chemin de résolution.
- Refresh : si un candidat élu est Unknown/absent du cache, UN
  refresh est programmé en tâche de fond (`tokio::spawn_blocking` :
  le connect TCP est bloquant, interdit dans la boucle UDP).
  Anti-spam : un refresh à la fois par cible (HashSet), relançable
  après la fin du refresh précédent.
- `TcpChecker` : connect TCP borné (timeout 2 s) vers
  `target:port`. Le **port ne peut pas être déduit d'un record DNS**
  : `--health-port` le fixe (défaut 443). Le checker est
  **volontairement naïf** (un connect réussi ≠ service sain) —
  améliorable (HTTP, TLS, applicatif) sans changement de format.
- TTL d'entrée : `--health-ttl-secs` (défaut 5 s) ; entrée périmée =
  Unknown = disponible.

### Sécurité

- Les records GEOROUTE sont des records signés comme les autres :
  vérifiés contre la chaîne (owner, séquence, record_hash) avant
  toute utilisation. **Aucune confiance réseau sans validation** —
  un GEOROUTE DHT non vérifié ne répond jamais.
- La position du client vient de la **configuration locale** du
  resolver (`--geo-country`/`--geo-continent`), jamais d'une API
  GeoIP réseau (pas de fuite de requête, pas de dépendance).
- Le tirage est déterministe et local : aucun secret, aucune
  imprévisibilité requise (la sécurité est celle de la signature du
  record, pas du tirage).

### Exemple complet (spec de référence)

`www.example.scone` publie (apex `example.scone`, tout le set sert
tout sous-nom) :

| Scope | Endpoint | Priorité | Poids |
|---|---|---|---|
| US (840) | 10.1.0.1 | 10 | 70 |
| US (840) | 10.1.0.2 | 10 | 30 |
| US (840) | 10.2.0.1 | 20 | 100 |
| TW (158) | 10.3.0.1 | 10 | 80 |
| TW (158) | 10.4.0.1 | 10 | 20 |
| TW (158) | 10.2.0.1 | 20 | 100 |
| WORLD | 10.0.0.1 | 100 | 100 |

Comportement attendu :

- client **US** → 10.1.0.1 (≈70 %) ou 10.1.0.2 (≈30 %) ; si les
  deux sont Unhealthy → failover p20 → **10.2.0.1** ;
- client **TW** → 10.3.0.1 (≈80 %) ou 10.4.0.1 (≈20 %) ;
- client **FR** (250) → pas de route FR ni EU dans le set →
  WORLD → **10.0.0.1** ;
- tout Unhealthy → pas de décision geo → NODATA (NOERROR, 0
  réponses) ;
- question **A** sur une route IPv6 only → réponse vide (jamais
  d'IPv6 en A) ;
- sans `--geo-dns` → même question, comportement classique
  (NODATA : le GEOROUTE n'a pas de rdata A wire).

### Options CLI (`scone relay`)

| Option | Défaut | Rôle |
|---|---|---|
| `--geo-dns` | off | Arme le GeoDNS du resolver (opt-in absolu) |
| `--geo-country <u16>` | — | Pays client, ISO 3166-1 numérique (840=US…) |
| `--geo-continent <u16>` | — | Continent client, UN M49 (150=EU…) ; dérivé du pays si omis |
| `--health-check` | off | Active les checks (failover santé) |
| `--health-ttl-secs <u64>` | 5 | TTL d'une entrée de santé |
| `--health-port <u16>` | 443 | Port TCP des checks (non déductible d'un record DNS) |

Exemple :

```bash
scone relay --dns 127.0.0.1:5353 \
            --geo-dns --geo-country 840 \
            --health-check --health-port 443
```

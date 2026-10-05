# Serveur DNS UDP + TCP (M6 + M7b) — `scone dns`

> Spécification normative de la surface DNS de Scone.
> Toute divergence doc/code = bug (corriger la source de vérité).

## Vue d'ensemble

Le **serveur DNS** est une surface optionnelle du relay : il répond
aux requêtes DNS **UDP et TCP** (RFC 7766, même port) pour les noms
Scone **uniquement à partir de données vérifiées contre la chaîne**,
et propose un **fallback récursif optionnel** (`addr:port`) pour le
reste. Il ne possède ni chaîne, ni store, ni DHT : les décisions
d'existence/expiration passent par le **snapshot DNS autoritaire**
(vue RAM publiée à chaque bloc appliqué — voir ci-dessous), et le
fetch des records passe par le relay (processus), via le canal
interne `RelayCommand`/`ResolveLocal` (in-process, un
aller-retour) — le même chemin vérifié que le CLI.

```text
              UDP+TCP :5353 (par défaut 127.0.0.1)
                    │
             scone relay --dns 127.0.0.1:5353
                    │  ┌────────────────────────────────┐
                    ├──┤ Cache shardé 16 voies (qname)   │
                    │  └────────────────────────────────┘
                    │  nom de forme Scone ?
                    │  ├─ OUI → snapshot DNS autoritaire :
                    │  │        inconnu/expiré → NXDOMAIN (RAM,
                    │  │        zéro aller-retour vers le cœur)
                    │  │        → recherche d'apex par suffixe
                    │  │        → `resolve_local` interne (relay)
                    │  │           chaîne + record vérifié (hash,
                    │  │           owner, séquence) → réponses
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

0. **Snapshot DNS autoritaire** : une vue immuable
   (`Arc<HashMap<DomainId, {valid_until, has_record}>>`, flux
   `tokio::sync::watch`) publiée par le cœur du relay **après chaque
   bloc appliqué** (push, pas poll) et au boot. Les décisions
   d'existence/expiration — NXDOMAIN, domaine expiré — sont servies
   depuis la RAM par ce snapshot : un clone d'`Arc` + un lookup de
   map, **sans jamais entrer dans le run loop mono-owner du relay**.
   Le chemin complet (`RelayCommand`/`ResolveLocal`) reste réservé
   au fetch effectif des records (pulls DHT), rares grâce au cache.
   Le snapshot ne porte JAMAIS de données de record : existence et
   échéance uniquement.
   - **Deltas O(touchés), copy-on-write** : la carte est un
     `Arc<HashMap>` ; une extension de tip publie le delta du bloc
     (les transactions du bloc nomment chaque domaine qu'elles
     peuvent changer — Register/Assign/Update/Renew/TransferDomain —
     plus les retraits du GC déterministe évalués au push) via
     `Arc::make_mut` : la carte n'est clonée QUE si un lecteur tient
     encore la vue précédente. Un boot ou une reorg reconstruit le
     snapshot complet (walk paginé du backend, une fois).
   - Fraîcheur : un lecteur peut voir un snapshot vieux de quelques
     centaines de ms pendant l'application d'un bloc — la même
     fraîcheur qu'un RPC de status ; les TTL du cache DNS ne sont
     pas affectés (les réponses restent autoritaires selon leur
     origine).
1. **Cache shardé** : [`ShardedCache`] — 16 shards de `Cache`
   indépendants, choisis par `hash(qname)` (le qtype partage le
   shard : localité par nom). Un nom chaud ne contend avec 1/16ᵉ du
   trafic au lieu de défiler derrière un mutex unique à plusieurs
   milliers de qps. Bornes GLOBALES (100 000 entrées / 64 MiB
   par défaut, `--dns-cache-limit` / `--dns-cache-max-mem`)
   réparties par shard (pire cas documenté : facteur 1 + N/shards).
   La taille RÉELLE des rdata de chaque entrée est comptabilisée
   (overhead par entrée ~128 o) et l'éviction FIFO se déclenche sur
   la première des deux bornes atteinte — le plafond mémoire tient
   même contre des entrées pleines (`MAX_DHT_RECORD_LEN`).
   *EN: hard byte budget tracks each entry's actual rdata size;
   FIFO evicts on entry-count or byte budget, whichever hits first.*
   TTL 60 s
   (autoritaire) / clampé à 600 s (fallback). Le cache négatif
   (NXDOMAIN/NODATA) suit les mêmes règles ; chaque entrée porte son
   origine (autoritaire ou fallback) pour que le bit AA reste honnête
   sur un hit. **Jamais de cache pour SERVFAIL** (défaillance locale
   transitoire) **ni REFUSED** (état de configuration) : recalculés à
   chaque requête. Éviction FIFO à la capacité.
   **Chemin chaud (inspiré Cloudflare/1.1.1.1)** :
   - entrées partagées `Arc<CacheEntry>` : un hit = un bump de
     refcount, JAMAIS un deep-clone des rdata (le `put` REMPLACE
     l'Arc — pas de mutation partagée ; le TTL est toujours vérifié
     sous le lock du shard) ;
   - lookup **zéro-allocation** : carte à deux niveaux
     (`Box<str>` qname → qtype → entrée), le `&str` emprunté de la
     requête probe directement (`Borrow<str>`) — l'ancienne clé
     `(String, u16)` allouait une `String` à CHAQUE lookup ;
   - **records packés contigus** : un seul buffer `Box<[u8]>` par
     entrée (`u16 rdlen ‖ u16 rtype ‖ u32 ttl ‖ rdata` par record)
     au lieu d'un `Vec` par record — le budget mémoire suit
     exactement ce qui est alloué ;
   - **compression de noms RFC 1035 en sortie** : l'owner des
     réponses autoritaires est un pointeur de compression 2 octets
     (`0xC00C`) vers la question recopiée — la question n'est jamais
     ré-encodée en entier par record.
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
4. **`resolve_local` (canal interne)** : un seul aller-retour, jamais
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
  réponses autoritaires avec **owner compressé RFC 1035** (pointeur
  2 octets `0xC00C` vers la question recopiée) ; les réponses du
  fallback amont sont servies telles quelles (jamais
  ré-encodées).
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
  locale), 3 NXDOMAIN (autoritaire), 5 REFUSED (hors Scone sans
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

## Cache négatif DHT (anti-amplification)

Un **pull DHT sans réponse** (record attendu on-chain —
`record_hash` engagé — mais absent du cache local et des pairs
interrogés) parque le nom pour `DHT_NEGATIVE_TTL_SECS` (30 s) dans
un cache négatif borné (FIFO, cap `DHT_NEGATIVE_CAP` = 4 096
noms) : les résolutions répétées du même nom dans le TTL sont
**supprimées** (comptées, jamais re-pullées). Un scanner — ou un
nom chaud dont l'owner n'a jamais publié — ne peut plus amplifier
une requête DNS en un `GetRecord` par pair du mesh et par requête.
L'entrée négative meurt au TTL, à l'arrivée du record (le pull
réussi écrit dans le vrai cache), ou poussée hors du cap FIFO.

Compteurs exposés (`scone_dht_negative_hits_total`,
`_expired_total`, `_capped_total` — voir « Métriques » dans
`relay.md`).

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
| Cache | bornes GLOBALES 100 000 entrées / 64 MiB par défaut, réparties sur 16 shards (`--dns-cache-limit` / `--dns-cache-max-mem`) |
| TTL autoritaire | **par enregistrement** (`record.ttl`, bornes 1..=2 592 000 s ; le 60 s constant historique ne reste que pour les réponses de service) |
| TTL fallback (clamp) | 600 s |
| Requêtes UDP en parallèle | 4 096 (`MAX_INFLIGHT_UDP`, sémaphore ; au-delà : drop, le client retente) — réglable `--dns-max-inflight` (min 1) |
| Cache négatif DHT | 30 s (`DHT_NEGATIVE_TTL_SECS`), cap FIFO 4 096 noms |
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
  (TXID ou question ne matchant pas), non-cache de SERVFAIL/REFUSED ;
  cache shardé : répartition des shards, bornes globales réparties,
  4 workers concurrents sur un nom chaud ; hot path : lookup
  zéro-allocation, entrées packées, compression d'owner.
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

## GeoDNS (routage géographique)

> Spécification normative. Toute divergence doc/code = bug.

### Table GeoIP embarquée (automatique)

Chaque binaire embarque une **table de correspondance IP → pays**
compilée dans le relay : **aucune configuration n'est nécessaire**,
le routage géographique est actif partout, automatiquement.

- **Source des données** : les fichiers de **délégations des RIR**
  (RIPE NCC, ARIN, APNIC, LACNIC, AFRINIC), librement
  téléchargeables et redistribuables :
  `https://ftp.ripe.net/pub/stats/<rir>/delegated-<rir>-extended-latest`
  (pays d'enregistrement de chaque bloc IPv4/IPv6 délégué).
  La date de génération figure dans l'en-tête de la table
  (`assets/embedded-geo.table`).
- **Régénération** : `scripts/gen-geo-table.py` (télécharge les 5
  fichiers, convertit, normalise les préfixes — ~176 000 règles).
- **Granularité** : pays (ISO 3166-1 numérique) + continent (UN
  M49) dérivé. Pas d'ASN ni de région (absents des délégations).
- **Performance** : lookup par **recherche binaire** sur plages
  triées (index construit une fois au premier usage, par famille
  v4/v6) — mesuré ~95 ns/lookup à chaud, ~50 µs en accès aléatoire
  sur la table complète (bench : `cargo test -p scone-network
  --release --test geo_bench -- --ignored`).
- **Auto-localisation du relay** : le routage se fait selon l'IP
  **source de chaque requête** — un relay sur un réseau local
  route ses clients locaux vers la bonne zone automatiquement.
- **Priorité** : une table opérateur explicite
  (`--geo-ip-table`/geo-config) **prime** sur l'embarquée.

### Vue d'ensemble

Un domaine peut publier des règles de **routage géographique** dans
son record set signé : des records `GEOROUTE` (type privé Scone
`32 768`, hors plage IANA). Le resolver répond aux questions
A/AAAA de ces domaines avec **l'adresse du endpoint choisi par la
règle** (scope le plus proche de la position du CLIENT, dérivée de
son IP source via la table), au lieu d'un record A/AAAA statique.
Tout le reste (domaines sans GEOROUTE, autres qtypes) suit le
chemin DNS classique, **octet pour octet**.

### Métriques GeoDNS

L'endpoint Prometheus expose la répartition des réponses geo :

- `scone_dns_geo_answers_total` — total des réponses GEOROUTE servies ;
- `scone_dns_geo_by_continent_total{continent="<M49>"}` — par continent ;
- `scone_dns_geo_by_country_total{country="<ISO>"}` — par pays
  (borné par les standards eux-mêmes : ≤ 9 continents, ≤ 250 pays).

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
           | 0x03 region.v    (subdivision ISO 3166-2 OPAQUE, u32 :
                                numéro défini par l'éditeur, comparé
                                tel quel — pas de table locale)
           | 0x04 asn.v       (numéro d'AS, u32)
           | 0x05 prefix.addr[16] || prefix.len u8
             (préfixe IP normalisé 16 octets : IPv4 = adresse dans
              les 4 premiers octets + queue À ZÉRO, len ≤ 32 ;
              IPv6 = 16 octets libres, len ≤ 128 ; bits d'hôte
              TOUJOURS nuls — un préfixe non normalisé est rejeté)
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

**Rétrocompatibilité wire (v1 → v2)** : les octets v1 sont
INCHANGÉS — les scopes v1 (monde/continent/pays) gardent leurs
discriminants `0x00..=0x02` et leur encodage exact ; les trois
nouveaux scopes prennent les discriminants suivants (`0x03..=0x05`)
dans un ordre total cohérent avec le tri canonique
(`World < Continent < Country < Region < Asn < IpPrefix`). Un
décodeur v1 qui rencontrait un discriminant ≥ `0x03` échouait déjà
strictement (rejet du record) ; les octets v1 produits restent
décodés à l'identique (voir le test `geo_known_bytes`, figé).

### Sélection (résolution)

1. **Scope** — spécificité décroissante, **6 niveaux** :
   **préfixe IP le plus long matchant → AS exact → pays exact →
   région exacte → continent → monde**.
   - *IpPrefix* : parmi les routes dont le préfixe matche l'IP du
     client (telle que connue du locator : `client_ip`, sinon les
     préfixes d'origine connus), la **plus longue** (`len` max)
     gagne ; à égalité de `len`, la première par ordre canonique
     (l'ordre du slice `routes`, trié à l'encodage). Une subdivision
     est moins spécifique qu'un préfixe/AS (qui identifient le
     réseau exact) et moins que le pays (numérotation opaque,
     fiable seulement si l'éditeur et le locator partagent la même),
     mais plus que le continent (elle est contenue dans son pays,
     donc a fortiori dans son continent).
   - *Asn* : numéro d'AS exact du client.
   - *Country* : pays exact.
   - *Region* : subdivision exacte (identifiant opaque).
   - *Continent* : fourni par le locator **ou dérivé du pays** par
     la table locale (`continent_of_country`) ; un pays inconnu de
     la table retombe sur le continent explicite ou le monde.
   - *World*.
   Une [`GeoLocation`] v1 (region/asn/prefixes/client_ip non
   renseignés) produit exactement le comportement v1 : pays →
   continent → monde. Aucune route qui matche (y compris sans scope
   World) → comportement DNS classique, comme si le GEOROUTE
   n'existait pas.
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
  resolver (`--geo-ip-table` ou `--geo-country`/`--geo-continent`/
  `--geo-asn`/`--geo-region`), jamais d'une API GeoIP réseau (pas de
  fuite de requête, pas de dépendance).
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
- domaine sans règle applicable → même question, comportement classique
  (NODATA : le GEOROUTE n'a pas de rdata A wire).

### Options CLI (`scone relay`)

| Option | Défaut | Rôle |
|---|---|---|
| (aucun — GeoDNS toujours actif, table embarquée) | on | Routage automatique par IP source |
| `--geo-ip-table <path>` | — | **Table GeoIP locale** (voir ci-dessous). PRIME sur toutes les options statiques `--geo-*` |
| `--geo-country <u16>` | — | Pays client, ISO 3166-1 numérique (840=US…) — statique |
| `--geo-continent <u16>` | — | Continent client, UN M49 (150=EU…) ; dérivé du pays si omis — statique |
| `--geo-asn <u32>` | — | AS client (64500…) — statique, nouveau v2 |
| `--geo-region <u32>` | — | Subdivision client (identifiant opaque éditeur) — statique, nouveau v2 |
| `--health-check` | off | Active les checks (failover santé) |
| `--no-health-check` | off | **Désactive explicitement** les checks — GAGNE sur `--health-check` (voir précédences) |
| `--health-ttl-secs <u64>` | 5 | TTL d'une entrée de santé |
| `--health-port <u16>` | 443 | Port TCP des checks (non déductible d'un record DNS) |

#### Table GeoIP locale (`--geo-ip-table`)

Fichier TEXTE local de préfixes, lu **une fois** au démarrage
(100 000 règles max, `MAX_GEOIP_RULES`). Une règle par ligne, `#` =
commentaire, lignes vides ignorées :

```text
# prefix/mask        pays  continent  asn     region
10.0.0.0/8           840   3          .       .
2001:db8::/32        250   150        64500   11
192.0.2.0/24         158   9
```

- `prefix/mask` : IPv4 (`len ≤ 32`, converti en forme 16 octets,
  queue à zéro — même convention que le wire) ou IPv6 (`len ≤ 128`).
  Les **bits d'hôte doivent être nuls** (`10.1.2.3/8` est refusé) ;
- `pays` : `u16` ISO 3166-1 **numérique**, requis ;
- `continent` : `u16` UN M49, optionnel (`.` ou omis = absent, le
  continent peut alors être dérivé du pays) ;
- `asn` : `u32`, optionnel (`.` = absent) ;
- `region` : `u32` opaque, optionnel (`.` = absent).

Résolution : l'**IP source de la requête DNS** (per-client :
UDP `recv_from` / TCP `peer_addr` ; si inconnue → IP de référence
du resolver) est comparée en 16 octets ; la règle au **plus long
préfixe matchant** gagne (à égalité, la première par ordre du
fichier) ; sans match → position vide (World).

**Strictesse** : le fichier est configuré par l'opérateur — une
ligne invalide (préfixe malformé, pays non numérique, continent
inconnu, champ en excès…) est une **erreur de démarrage** portant le
**numéro de la ligne fautive** ; jamais de ligne ignorée en silence,
jamais de table partielle.

#### Précédences des options

- **Locator** : `--geo-ip-table` **prime** sur les options statiques
  (`--geo-country`/`--geo-continent`/`--geo-asn`/`--geo-region`),
  qui sont ignorées quand la table est fournie. Sans table, les
  options statiques s'appliquent.
- **Health** : `--no-health-check` est un **drapeau dédié qui gagne**
  sur `--health-check` — la combinaison `--health-check
  --no-health-check` laisse les checks désactivés, sans ambiguïté
  d'ordre de saisie. Sans `--health-check`, les checks sont off par
  défaut : **Unknown = disponible**, la réponse geo part même si la
  cible est morte. (Un `NoopChecker` — toujours Healthy — existe
  aussi pour monter la structure health sans écarter aucun
  endpoint : tests/paranoïa.)

Exemple :

```bash
scone relay --dns 127.0.0.1:5353 \
            --geo-ip-table /etc/scone/geoip.txt \
            --no-health-check
```

Exemple statique v2 :

```bash
scone relay --dns 127.0.0.1:5353 \
            --geo-country 250 --geo-asn 64500 --geo-region 11 \
            --health-check --health-port 443
```


## DoH / DoT (transports chiffrés) — via frontal externe

Le serveur DNS de Scone parle **UDP et TCP classiques** (RFC 1035 /
7766) — c'est le périmètre natif, complet et testé. Les transports
chiffrés vers le *client final* (DoH — DNS-over-HTTPS, RFC 8484 ;
DoT — DNS-over-TLS, RFC 7858) sont **délégués à un frontal
standard** : Scone sert les données, le frontal sert le transport.

Pourquoi ce choix : un serveur DoH natif exigerait une pile HTTPS
complète dans le nœud (certificats, renouvellement, HTTP/2) — une
surface opérationnelle importante pour un bénéfice nul avant qu'il
y ait des clients finaux navigateur. Les frontaux existants
gèrent ça mieux et depuis plus longtemps ; le nœud reste
minimaliste.

### Exemple : unbound en DoH devant un relay Scone

`/etc/unbound/unbound.conf` :

```
server:
    interface: 127.0.0.1
    port: 5354
    do-ip4: yes
    do-udp: no
    do-tcp: yes

    # DoH (RFC 8484) sur 443
    https-port: 443
    tls-service-key: "/etc/unbound/scone.key"
    tls-service-pem: "/etc/unbound/scone.pem"

forward-zone:
    name: "."
    forward-addr: 127.0.0.1@5353   # le relay Scone (--dns 127.0.0.1:5353)
```

Le certificat (`scone.pem`/`scone.key`) est obtenu comme pour
n'importe quel service TLS — par exemple via certbot/acme.sh sur le
domaine du resolver. Les clients DoH pointent alors sur
`https://dns.exemple.tld/dns-query`.

### Exemple : knot-resolver en DoT

`/etc/knot-resolver/kresd.conf` :

```
net.listen('127.0.0.1', 5353, { kind = 'dns' })
net.listen('0.0.0.0', 853, { kind = 'tls' })
tls.cert_file = '/etc/knot-resolver/scone.pem'
tls.key_file  = '/etc/knot-resolver/scone.key'
```

### Ce que le frontal ajoute

- **Chiffrement du canal client→frontal** (le segment hostile :
  le réseau de l'utilisateur) ; le segment frontal→relay reste
  local (même machine ou réseau de confiance).
- **Cache supplémentaire côté frontal** (unbound/knot cachent
  aussi — inoffensif, les TTL Scone restent la limite haute).
- **Point de terminaison stable** (IPv4/IPv6, domaine) pendant que
  les relays bougent.

### Ce que le frontal NE fait PAS

- Il ne *vérifie* rien : toute donnée servie par le relay est déjà
  vérifiée contre la chaîne avant d'être cachée (voir « Chemin de
  résolution »). Le frontal relaie des réponses déjà autoritaires.
- Il ne remplace pas la décentralisation : un frontal DoH est un
  service *opéré* — c'est le choix de l'opérateur du resolver
  public, pas du protocole. Chaque relay Scone reste
  indépendamment utilisable en UDP/TCP natif.

### Choix d'implémentation future (documenté, non engagé)

Un transport DoH natif (`--doh ADDR`) resterait envisageable :
le format wire étant identique (RFC 8484 réutilise les messages
DNS binaires), il s'agirait d'un décodage POST/GET + HTTP/2 derrière
un proxy TLS (caddy/nginx gérant les certificats), le nœud restant
sans TLS intégré. Aucun travail en cours — la demande arrivera avec
les clients finaux.

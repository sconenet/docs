# Serveur DNS UDP + TCP (M6 + M7b) — `scone dns`

> Spécification normative de la surface DNS de Scone.
> Toute divergence doc/code = bug (corriger la source de vérité).

## Vue d'ensemble

Le **serveur DNS** est une surface optionnelle du relais : il répond
aux requêtes DNS **UDP et TCP** (RFC 7766, même port) pour les noms
Scone **uniquement à partir de données vérifiées contre la chaîne**,
et offre un **fallback récursif optionnel** (`addr:port`) pour tout le
reste. Il ne possède ni chaîne, ni store, ni DHT : les décisions
d'existence/expiration passent par le **snapshot DNS autoritaire**
(une vue RAM publiée après chaque bloc appliqué — voir ci-dessous), et
les récupérations d'enregistrements passent par le relais (processus),
via le canal interne `RelayCommand`/`ResolveLocal` (in-process, un
aller-retour) — le même chemin vérifié que le CLI.

```text
              UDP+TCP :5353 (default 127.0.0.1)
                    │
             scone relay --dns 127.0.0.1:5353
                    │  ┌────────────────────────────────┐
                    ├──┤ Sharded cache, 16 ways (qname)  │
                    │  └────────────────────────────────┘
                    │  Scone-shaped name?
                    │  ├─ YES → authoritative DNS snapshot:
                    │  │        unknown/expired → NXDOMAIN (RAM,
                    │  │        zero round trips into the core)
                    │  │        → apex search by suffix
                    │  │        → internal `resolve_local` (relay)
                    │  │           chain + verified record (hash,
                    │  │           owner, sequence) → answers
                    │  │        apex without a published record → NODATA
                    │  └─ NO → upstream fallback (if configured)
                    │           otherwise REFUSED
                    │
                    │  response budget:
                    │    UDP → 512 bytes (no EDNS); overflow =
                    │          cut at the last fitting record + TC=1 →
                    │          the client retries over TCP
                    │    TCP → 64 KiB (2-byte length prefix,
                    │          RFC 1035 §4.2.2): full set
```

## Démarrage

```bash
scone relay --dns 127.0.0.1:5353            # DNS surface only
scone relay --dns 0.0.0.0:53 \
            --dns-upstream 1.1.1.1:53 \
            --dns-upstream 8.8.8.8:53       # + recursive fallback
```

Le relais annonce l'adresse effective sur stdout (contrat
lisible par machine, comme `p2p:`) : une ligne `dns: <addr>` dès que
le socket est lié.

Client de validation : `dig <name> --dns 127.0.0.1:5353
--qtype A` (A, AAAA, TXT, MX, NS, CNAME, ANY).

## Chemin de résolution (AUTORITÉ = la chaîne, toujours)

0. **Snapshot DNS autoritaire** : une vue immuable
   (`Arc<HashMap<DomainId, {valid_until, has_record}>>`,
   flux `tokio::sync::watch`) publiée par le cœur du relais **après
   chaque bloc appliqué** (push, pas poll) et au démarrage. Les
   décisions d'existence/expiration — NXDOMAIN, domaine expiré — sont
   servies depuis la RAM via ce snapshot : un clone d'`Arc` + une
   recherche dans la map, **sans jamais entrer dans la boucle
   mono-propriétaire du relais**. Le chemin complet
   (`RelayCommand`/`ResolveLocal`) reste réservé aux récupérations
   d'enregistrements réelles (pulls DHT), rares grâce au cache. Le
   snapshot ne transporte JAMAIS de données d'enregistrement :
   existence et expiration uniquement.
   - **Deltas copy-on-write en O(touchés)** : la map est une
     `Arc<HashMap>` ; une extension de sommet publie le delta du bloc
     (les transactions du bloc nomment elles-mêmes chaque domaine
     qu'elles peuvent changer — Register/Assign/Update/Renew/
     TransferDomain — plus les retraits deterministes du GC
     d'expiration évalués au moment du push) via `Arc::make_mut` : la
     map n'est clonée QUE si un lecteur détient encore la vue
     précédente. Le démarrage et les réorgs reconstruisent le
     snapshot complet (un parcours paginé du backend).
   - Fraîcheur : un lecteur peut voir un snapshot âgé de quelques
     centaines de ms durant l'application d'un bloc — la même
     désuétude qu'aurait un RPC de statut ; les TTL du cache DNS ne
     sont pas affectés (les réponses restent autoritaires selon leur
     origine).
1. **Cache shardingé** : [`ShardedCache`] — 16 fragments `Cache`
   indépendants, clés par `hash(qname)` (le qtype partage le
   fragment : localité par nom). Un nom chaud se dispute 1/16e du
   trafic au lieu de faire la queue derrière un mutex unique à des
   milliers de qps. Bornes GLOBALES (100 000 entrées / 64 MiB par
   défaut, `--dns-cache-limit` / `--dns-cache-max-mem`) réparties par
   fragment (pire cas documenté : un facteur 1 + N/shards). La taille
   RÉELLE des rdata de chaque entrée est comptabilisée (~128 octets
   de surcoût par entrée) et l'éviction FIFO se déclenche à la
   première des deux bornes atteinte — le plafond mémoire tient même
   face à des entrées pleine taille (`MAX_DHT_RECORD_LEN`). TTL
   60 s (autoritaire) / borné à 600 s (fallback). Le cache négatif
   (NXDOMAIN/NODATA) suit les mêmes règles ; chaque entrée porte son
   origine (autoritaire ou fallback) pour que le bit AA reste honnête
   sur un hit. **Ne jamais mettre en cache SERVFAIL** (défaillance
   locale transitoire) **ni REFUSED** (état de configuration) : les
   deux sont recalculés à chaque requête. Éviction FIFO à capacité.
   **Chemin chaud (inspiré Cloudflare/1.1.1.1)** :
   - entrées `Arc<CacheEntry>` partagées : un hit est un incrément de
     compteur de références, JAMAIS un clonage profond des rdata
     (`put` REMPLACE l'Arc — pas de mutation partagée ; le TTL reste
     vérifié sous le verrou du fragment) ;
   - **recherche sans allocation** : map à deux niveaux
     (`Box<str>` qname → qtype → entrée), le `&str` emprunté de la
     requête sonde directement (`Borrow<str>`) — l'ancienne clé
     tuple `(String, u16)` allouait une `String` fraîche à CHAQUE
     recherche ;
   - **enregistrements contigus empaquetés** : un tampon `Box<[u8]>`
     par entrée (`u16 rdlen ‖ u16 rtype ‖ u32 ttl ‖ rdata` par
     enregistrement) au lieu d'un `Vec` par enregistrement — le
     budget mémoire suit exactement ce qui est alloué ;
   - **compression de noms RFC 1035 en sortie** : le propriétaire des
     réponses autoritaires est un pointeur de compression de 2 octets
     (`0xC00C`) vers la question échotée — la question n'est jamais
     ré-encodée intégralement par enregistrement.
2. **Forme Scone ?** Pré-filtre strict : labels `[a-z0-9-_]`
   (1..63), ≥ 2 labels, TLD 1..63 octets `[a-z0-9-]` (voir
   `TldName::MAX_LEN`), longueur totale ≤ 253 — **et TLD hors de la
   racine ICANN** (`is_icann_tld`, liste IANA gelée + RFC 2606 :
   `com`, `fr`, `org`… n'appartiennent jamais à Scone). Un nom qui ne
   PEUT PAS être un nom Scone (p. ex. `www.foo_bar`, underscore dans
   le TLD) va au fallback ; un nom Scone possible (`absent.uip`)
   regarde la chaîne : non enregistré → NXDOMAIN autoritaire, même
   sans upstream. Un TLD ICANN (`example.com`) est transmis aux
   upstreams si configurés, REFUSED sinon — jamais résolu de façon
   autoritaire par Scone.
3. **Recherche d'apex** : suffixes du plus long au plus court (min.
   2 labels). Le premier suffix dont l'état on-chain est connu du
   relais = apex ; l'ensemble d'enregistrements sert le qname entier
   (sous-noms compris — comportement wildcard implicite, décision
   ouverte).
4. **`resolve_local` (canal interne)** : un seul aller-retour, jamais
   d'attente DHT. État on-chain (`registered`, owner, sequence) +
   enregistrement du cache DHT local **revérifié** (owner, sequence,
   record_hash == engagement on-chain). Un enregistrement non
   conforme ne répond jamais — disponibilité ≠ autorité.
   - non enregistré → NXDOMAIN ;
   - enregistré sans enregistrement valide → NODATA (NOERROR, 0
     réponses) ;
   - enregistrement vérifié → réponses filtrées par qtype.

## Codec (RFC 1035, sous-ensemble strict)

- Requête : en-tête de 12 octets, **une** question, QCLASS = IN,
  **pas de compression** (pointeur = rejet). Labels passés en
  minuscules et validés. QR=1, QDCOUNT≠1, labels invalides →
  **silence** (les déchets ne coûtent rien).
- Réponse : QR|RD(echo)|RA, **AA uniquement sur les réponses
  autoritaires** (chaîne, fraîche ou en cache — jamais sur le
  fallback ni les RCODE d'erreur), question copiée verbatim,
  réponses autoritaires avec un **propriétaire compressé RFC 1035**
  (pointeur 2 octets `0xC00C` vers la question échotée) ; les
  réponses upstream du fallback sont servies telles quelles (jamais
  ré-encodées).
- Types servis : A, AAAA, CNAME, NS, MX, TXT, **PTR, SRV, CAA**
  (+ Unknown ré-encodé verbatim ; 9 types natifs au total, rdata
  encodés par RFC — 1035/2782/8659 ; CAA strict : bit critique 0x80
  uniquement, tags en minuscules). Une requête A/AAAA inclut les
  CNAME de l'ensemble (poursuite d'alias). Requête ANY (255) :
  l'ensemble entier filtré par le codec.
- Limites (M7b) : **le budget de réponse dépend du transport** —
  512 octets en UDP (charge classique RFC 1035 §2.3.4, pas d'EDNS),
  65 535 en TCP. Un ensemble dépassant le budget est coupé au
  dernier enregistrement qui tient, avec **TC=1** (troncature honnête,
  RFC 1035 §4.2.1) : le client réessaie en TCP (RFC 7766), où
  l'ensemble complet est servi. Une réponse upstream plus grande que
  le budget UDP est ré-encodée tronquée (TC=1) au lieu d'être livrée
  telle quelle ; en TCP elle passe octet pour octet. La troncature
  s'applique aussi aux hits de cache (le budget est appliqué au
  moment de la réponse, pas au remplissage du cache).
- RCODE : 0 NOERROR (avec ou sans réponses), 2 SERVFAIL (défaillance
  locale), 3 NXDOMAIN (autoritaire), 5 REFUSED (non-Scone sans
  upstream).

## Fallback récursif

- Upstreams `addr:port` (UDP), essayés dans l'ordre, délai d'attente
  de 2 s par upstream, socket **connecté** à l'upstream (filtrage
  noyau : un paquet venant de toute autre source n'est jamais
  accepté). La réponse upstream n'est acceptée que si elle
  **correspond à la requête** : même TXID, QR=1, question
  (QNAME+QTYPE+QCLASS) copiée octet pour octet. Une réponse non
  conforme → SERVFAIL, et rien n'est mis en cache.
- La réponse upstream validée passe telle quelle ; ses réponses sont
  mises en cache avec l'origine `fallback` (AA jamais positionné,
  même sur un hit de cache) et TTL borné à 600 s. Les RCODE d'erreur
  upstream (autres que NOERROR/NXDOMAIN) ne sont pas mis en cache.
- **Privé par défaut** : pas d'upstream = REFUSED pour les noms hors
  forme Scone.

## Cache négatif DHT (anti-amplification)

Un **pull DHT sans réponse** (enregistrement attendu on-chain —
`record_hash` engagé — mais absent du cache local et des pairs
interrogés) parque le nom pour `DHT_NEGATIVE_TTL_SECS` (30 s) dans un
cache négatif borné (FIFO, plafond `DHT_NEGATIVE_CAP` = 4 096 noms) :
les résolutions répétées du même nom dans le TTL sont **supprimées**
(comptées, jamais re-pullées). Un scanner — ou un nom chaud dont le
propriétaire n'a jamais publié — ne peut plus amplifier une requête
DNS en un `GetRecord` par pair du maillage par requête. L'entrée
négative meurt au TTL, à l'arrivée d'un enregistrement (le pull
réussi écrit dans le vrai cache) ou en étant poussée hors du plafond
FIFO.

Compteurs exposés (`scone_dht_negative_hits_total`,
`_expired_total`, `_capped_total` — voir « Métriques » dans
`relay.md`).

## TCP (M7b, RFC 7766)

Même port qu'UDP (second bind TCP sur l'adresse `--dns`). Le
transport suit la RFC 1035 §4.2.2 : chaque message est préfixé par sa
longueur sur **2 octets** gros-boutiste.

- **Requête** : longueur du préfixe ≥ 1 et ≤ 512 octets (une requête
  DNS tient dans la charge classique — RFC 7766 §6 ; pas d'EDNS
  servi). Hors bornes → fermeture immédiate de la connexion.
- **Réponse** : budget de 65 535 octets (maximum du préfixe 2 octets)
  — un ensemble trop grand pour l'UDP est servi **en entier** ici,
  TC=0.
- **Pipelining** : plusieurs requêtes par connexion sont servies en
  série, dans l'ordre (une écriture = une réponse préfixée).
- **Délai de lecture** : 5 s par lecture (préfixe ou corps) — un
  client silencieux est déconnecté, son créneau libéré.
- **Connexions concurrentes** : 64 max (sémaphore) ; la 65e est
  rejetée immédiatement (connexion fermée), jamais mise en file —
  aucune croissance non bornée de tâches ou de mémoire.
- Erreurs de lecture/écriture/délai : log debug + fermeture propre.
  **Jamais de panic sur une entrée réseau.**
- Les déchets (requête incompréhensible) reçoivent le silence mais ne
  ferment pas la connexion : la requête suivante (valide) est servie.

```text
client                     server
  │ [len u16][query]         │
  │ ─────────────────────►   │ cache → authoritative → fallback
  │ ◄─────────────────────   │ [len u16][response ≤ 64 KiB]
  │ (UDP had answered TC=1 → the client retries here)
```

## Bornes et garde-fous

| Garde-fou | Valeur |
|---|---|
| Datagramme UDP (requête/réponse) | 4096 octets (rejeté au-delà) |
| Budget de réponse UDP (TC=1 au-delà) | 512 octets (pas d'EDNS) |
| Requête TCP max | 512 octets (fermeture au-delà) |
| Réponse TCP max | 65 535 octets |
| Connexions TCP concurrentes | 64 (sémaphore, rejet immédiat) |
| Délai de lecture TCP | 5 s |
| Cache | bornes GLOBALES 100 000 entrées / 64 MiB par défaut, réparties sur 16 fragments (`--dns-cache-limit` / `--dns-cache-max-mem`) |
| TTL autoritaire | **par enregistrement** (`record.ttl`, bornes 1..=2 592 000 s ; la constante historique 60 s ne reste que pour les réponses de service) |
| TTL fallback (borne) | 600 s |
| Requêtes UDP parallèles | 4 096 (`MAX_INFLIGHT_UDP`, sémaphore ; au-delà : drop, le client réessaie) — réglable `--dns-max-inflight` (min 1) |
| Cache négatif DHT | 30 s (`DHT_NEGATIVE_TTL_SECS`), plafond FIFO 4 096 noms |
| Délai par upstream | 2 s |

Règles projet respectées : jamais de panic sur des données réseau ;
décodage strict partout ; le serveur ne touche ni la chaîne, ni le
store, ni le DHT (le relais reste la seule autorité de vérification).

## Tests

- Unitaires (`scone-network::dns`) : codec strict (QR, compression,
  rejets de labels), réponses autoritaires, NODATA, NXDOMAIN, REFUSED
  sans upstream, fallback vers un mock UDP, cache borné,
  découpage TXT, parsage upstream ; correctif M6 : honnêteté du bit
  AA (autoritaire vs fallback vs erreurs, y compris hits de cache),
  troncature TC=1 d'un ensemble trop grand, rejet de réponse upstream
  (TXID ou question divergente), pas de mise en cache de
  SERVFAIL/REFUSED ; cache shardingé : répartition des fragments,
  bornes globales réparties, 4 workers concurrents sur un nom chaud ;
  chemin chaud : recherche sans allocation, entrées empaquetées,
  compression du propriétaire.
- M7b : budget par transport (UDP 512 octets → TC=1, TCP → ensemble
  complet), troncature d'un hit de cache, vrai `run_tcp` (ensemble >
  512 servi en entier, pipelining de 2 requêtes, requête hors bornes →
  fermeture, client silencieux → délai 5 s, déchets puis requête
  valide sur la même connexion), réponse upstream > 512 ré-encodée
  TC=1 en UDP / octet pour octet en TCP.
- Intégration réelle (`tests/dns_server.rs`) : un relais complet
  (devnet de production compris) + enregistrement + UpdateDomain +
  PutRecord, puis vraies requêtes UDP : A vérifié (bit AA), TXT,
  NODATA AAAA, NXDOMAIN, fallback via mock, re-requête, sous-nom via
  apex.
- E2E binaire (`crates/scone/tests/dns_cli.rs`) : le vrai binaire
  `scone` — `relay --dns`, `identity generate`, `domain register`,
  `domain update --file`, puis `dig` (dnsutils) (A vérifié, TXT,
  NXDOMAIN, REFUSED) sur un vrai port UDP.

## Décisions ouvertes

- Pas d'EDNS (le budget UDP reste 512 octets ; la troncature TC=1 +
  TCP couvre le besoin devnet).
- L'ensemble d'enregistrements ne porte pas de nom par
  enregistrement : l'ensemble entier sert chaque sous-nom de l'apex
  (wildcard implicite). Un adressage par enregistrement exigerait
  d'étendre `DnsRecord` (protocole).
- Le fallback transmet la requête brute : un upstream malveillant
  peut répondre n'importe quoi (réponse non validée — c'est le modèle
  récursif standard ; les noms Scone ne passent JAMAIS par lui).

## GeoDNS (routage géographique)

> Spécification normative. Toute divergence doc/code = bug.

### Table GeoIP embarquée (automatique)

Chaque binaire embarque une **table de correspondance IP → pays**
compilée dans le relais : **aucune configuration nécessaire**, le
routage géographique est actif partout, automatiquement.

- **Source de données** : **DB-IP Lite** (« IP to Country Lite »),
  licence **CC BY 4.0** — redistribuable avec attribution :
  `https://db-ip.com` (curée mensuellement ; auparavant : les
  délégations RIR via `scripts/gen-geo-table.py`, retiré).
  Cellules quantizées **/20 v4 et /24 v6** (la granularité native
  DB-IP, ~1,3 M de plages, dépasse la borne mémoire du parseur) —
  ~96 500 règles. La date de génération est dans l'en-tête de la
  table (`assets/embedded-geo.table`).
- **Checksum** : le SHA-256 de la table est calculé à la
  compilation (`build.rs`), **revérifié au démarrage** (mismatch =
  refus typé — un binaire altéré ne sert jamais de réponses
  géographiques) et journalisé au boot avec l'attribution. Le hash
  est la garantie que « tous les nœuds d'un même build partagent la
  même table ».
- **Régénération** : `scripts/geo-convert.py` (télécharge le CSV
  DB-IP Lite du mois, convertit — pays numérique, continent dérivé
  ISO→M49, ASN/région absents de la Lite).
- **Granularité** : pays (ISO 3166-1 numérique) + continent (ONU
  M49) dérivé. Pas d'ASN ni de région (absents de la Lite — les
  tables opérateur `--geo-ip-table` les couvrent).
- **Performance** : recherche par **dichotomie** sur des plages
  triées (index construit une fois à la première utilisation, par
  famille v4/v6) — mesuré ~95 ns/recherche à chaud, ~50 µs en accès
  aléatoire sur la table complète (bench : `cargo test -p
  scone-network --release --test geo_bench -- --ignored`).
- **Auto-localisation du relais** : le routage suit l'**IP source de
  chaque requête** — un relais LAN route ses clients locaux vers la
  bonne zone automatiquement.
- **Table IMPOSÉE** (remove-geo-operator-table, 2026-10-22) : la
  table embarquée est la **seule source** — aucun import ni surcharge
  opérateur n'existe. Le flag hérité `--geo-ip-table` et la clé
  `ip_table` de geo-config sont acceptés **sans effet** (comme
  `--rpc`). Un opérateur avec des besoins spécifiques (ASN, région)
  attend une évolution du pipeline embarqué, pas une table locale.

### Vue d'ensemble

Un domaine peut publier des règles de **routage géographique** dans
son ensemble d'enregistrements signés : des enregistrements
`GEOROUTE` (type privé Scone `32 768`, hors de la plage IANA). Le
résolveur répond aux questions A/AAAA pour ces domaines avec
**l'adresse du point de terminaison choisi par la règle** (portée la
plus proche de la position du CLIENT, dérivée de son IP source via la
table), au lieu d'un enregistrement A/AAAA statique. Tout le reste
(domaines sans GEOROUTE, autres qtypes) suit le chemin DNS classique,
**octet pour octet**.

### Métriques GeoDNS

Le point de terminaison Prometheus expose la répartition des réponses
geo :

- `scone_dns_geo_answers_total` — total des réponses GEOROUTE
  servies ;
- `scone_dns_geo_by_continent_total{continent="<M49>"}` — par
  continent ;
- `scone_dns_geo_by_country_total{country="<ISO>"}` — par pays
  (borné par les normes elles-mêmes : ≤ 9 continents, ≤ 250 pays).

### Enregistrement GEOROUTE (wire)

Un GEOROUTE = une **portée** + ≤ 32 **points de terminaison** (tri
canonique au décodage : priorité croissante, poids décroissant,
cible croissante ; doublon = rejet) :

```text
GeoScope   = 0x00 (world)
           | 0x01 continent.v (UN M49 code: AF=2, AS=142, EU=150,
                                NA=3, SA=5, OC=9, AN=999)
           | 0x02 country.v   (ISO 3166-1 numeric code: US=840,
                                TW=158, FR=250…)
           | 0x03 region.v    (OPAQUE ISO 3166-2 subdivision, u32:
                                publisher-defined number, compared
                                as is — no local table)
           | 0x04 asn.v       (AS number, u32)
           | 0x05 prefix.addr[16] || prefix.len u8
             (normalized 16-byte IP prefix: IPv4 = address in the
              first 4 bytes + ZERO tail, len ≤ 32;
              IPv6 = 16 free bytes, len ≤ 128; host bits ALWAYS
              zero — a non-normalized prefix is rejected)
GeoTarget  = 0x00 octets[4]  (binary IPv4 address)
           | 0x01 octets[16] (binary IPv6 address)
GeoEndpoint = target || priority.v || weight.v
GEOROUTE    = scope || count.v(≤32) || endpoints…
```

Bornes de validation (décodage) : `priority > 0`, `weight ≤ 100`
(`weight = 0` = point de terminaison inactif, jamais élu), TTL dans
1..=2 592 000 s comme tout enregistrement. Un pays/continent inconnu
des tables locales est **permis** sur le wire : le routage retombe
simplement sur une portée supérieure.

**Compatibilité wire rétroactive (v1 → v2)** : les octets v1 sont
INCHANGÉS — les portées v1 (monde/continent/pays) gardent leurs
discriminants `0x00..=0x02` et leur encodage exact ; les trois
nouvelles portées prennent les discriminants suivants (`0x03..=0x05`)
dans un ordre total cohérent avec le tri canonique
(`World < Continent < Country < Region < Asn < IpPrefix`). Un décodeur
v1 rencontrant un discriminant ≥ `0x03` échouait déjà strictement
(enregistrement rejeté) ; les octets produits en v1 restent décodés à
l'identique (voir le test `geo_known_bytes`, gelé).

### Sélection (résolution)

1. **Portée** — spécificité décroissante, **6 niveaux** : **plus long
   préfixe IP correspondant → AS exact → pays exact → région exacte →
   continent → monde**.
   - *IpPrefix* : parmi les routes dont le préfixe correspond à l'IP
     du client (telle que connue du localisateur : `client_ip`, sinon
     les préfixes d'origine connus), la plus **longue** (`len` max)
     gagne ; à `len` égal, la première dans l'ordre canonique
     (l'ordre de la tranche `routes`, triée à l'encodage). Une
     subdivision est moins spécifique qu'un préfixe/AS (qui
     identifient le réseau exact) et moins que le pays (numérotation
     opaque, fiable seulement si l'éditeur et le localisateur
     partagent la même), mais plus que le continent (elle est
     contenue dans son pays, donc a fortiori dans son continent).
   - *Asn* : numéro d'AS exact du client.
   - *Country* : pays exact.
   - *Region* : subdivision exacte (identifiant opaque).
   - *Continent* : fourni par le localisateur **ou dérivé du pays**
     via la table locale (`continent_of_country`) ; un pays inconnu
     de la table retombe sur le continent explicite ou le monde.
   - *World*.
   Une [`GeoLocation`] v1 (region/asn/prefixes/client_ip non
   renseignés) produit exactement le comportement v1 : pays →
   continent → monde. Aucune route correspondante (y compris sans
   portée World) → comportement DNS classique, comme si le GEOROUTE
   n'existait pas.
2. **Priorité** — sémantique SRV : seul le groupe de **priorité
   minimale** est éligible ; les priorités supérieures sont des
   niveaux de **bascule** (failover).
3. **Poids** — dans le groupe élu, tirage **pondéré déterministe** :
   hachage stable (SplitMix64) d'un nonce (le compteur de requêtes du
   résolveur) contre les poids cumulés. Pas de RNG : deux requêtes
   avec le même nonce donnent le même résultat, des nonces croissants
   répartissent proportionnellement aux poids. `weight = 0` =
   inactif, jamais élu (même seul — passage au niveau suivant).
4. **Bascule de santé** — un point de terminaison **Unhealthy** est
   mis de côté ; si tout le niveau minimal est indisponible, passage
   au niveau de priorité suivant. **Inconnu compte comme disponible**
   (l'inconnu n'est pas pénalisé). Tout indisponible → comportement
   DNS classique.

### Réponse

- Une seule adresse : **A si IPv4, AAAA si IPv6** — filtrage strict
  par famille selon le qtype (une question A ne répond JAMAIS en
  IPv6, et inversement).
- TTL = celui de l'enregistrement GEOROUTE source (borné au décodage
  par les règles générales de TTL).
- AA positionné (réponse autoritaire — le GEOROUTE est un
  enregistrement signé vérifié comme les autres).
- Réponse vide après filtrage (aucun candidat de la bonne famille,
  tout Unhealthy, aucune route correspondante) → le résolveur
  poursuit sur son chemin classique (typiquement NODATA : NOERROR, 0
  réponses) ; les rcodes d'erreur suivent les règles générales
  (jamais mis en cache).
- Un enregistrement A/AAAA classique coexistant dans l'ensemble
  PRIME sur le geo (le geo ne s'applique qu'en l'absence de réponse
  wire pour le qtype).

### Cache : décision explicite

La décision de routage geo n'est **JAMAIS mise en cache** sous
`(qname, qtype)` : elle dépend de la portée client du résolveur, et
une telle clé servirait la route US à un client TW. Les réponses geo
sont recalculées à chaque requête (coût : un select_scope + tirage
pondéré, µs) et contournent le cache. Le cache classique (qname,
qtype) reste inchangé pour tout le reste — y compris le NODATA produit
quand le geo ne décide rien. Les rcodes d'erreur restent jamais mis
en cache.

### Vérifications de santé (`--health-check`)

- `HealthCache` par cible (`GeoTarget`) : `probe` = lecture non
  bloquante (statut frais, ou Unknown), **jamais** une vérification
  déclenchée depuis le chemin de résolution.
- Rafraîchissement : si un candidat élu est Unknown/absent du cache,
  UN rafraîchissement est planifié en tâche de fond
  (`tokio::spawn_blocking` : le connect TCP est bloquant, interdit
  dans la boucle UDP). Anti-spam : un rafraîchissement à la fois par
  cible (HashSet), redémarrable après la fin du rafraîchissement
  précédent.
- `TcpChecker` : connect TCP borné (délai 2 s) vers `target:port`.
  Le **port ne peut pas se déduire d'un enregistrement DNS** :
  `--health-port` le définit (défaut 443). Le vérificateur est
  **délibérément naïf** (un connect réussi ≠ service sain) —
  améliorable (HTTP, TLS, niveau applicatif) sans changement de
  format.
- TTL d'entrée : `--health-ttl-secs` (défaut 5 s) ; entrée expirée =
  Unknown = disponible.

### Sécurité

- Les enregistrements GEOROUTE sont des enregistrements signés comme
  les autres : vérifiés contre la chaîne (owner, sequence,
  record_hash) avant tout usage. **Aucune confiance réseau sans
  validation** — un GEOROUTE DHT non vérifié ne répond jamais.
- La position du client vient de la **table GeoIP embarquée**
  (DB-IP Lite, voir ci-dessus — imposée par le build) dérivée de son
  IP source — ou de la déclaration statique du relais
  (`--geo-country`/`--geo-continent`/`--geo-asn`/`--geo-region`,
  pour un relais LAN sans IP publique résolvable). Aucun import de
  table IP opérateur n'existe (les geozones sont imposées par le
  build), jamais d'une API GeoIP réseau (aucune fuite de
  requête, aucune dépendance).
- Le tirage est déterministe et local : pas de secret, aucune
  imprévisibilité requise (la sécurité est celle de la signature de
  l'enregistrement, pas du tirage).

### Exemple complet (spécification de référence)

`www.example.scone` publie (apex `example.scone`, l'ensemble entier
sert chaque sous-nom) :

| Portée | Point de terminaison | Priorité | Poids |
|---|---|---|---|
| US (840) | 10.1.0.1 | 10 | 70 |
| US (840) | 10.1.0.2 | 10 | 30 |
| US (840) | 10.2.0.1 | 20 | 100 |
| TW (158) | 10.3.0.1 | 10 | 80 |
| TW (158) | 10.4.0.1 | 10 | 20 |
| TW (158) | 10.2.0.1 | 20 | 100 |
| WORLD | 10.0.0.1 | 100 | 100 |

Comportement attendu :

- client **US** → 10.1.0.1 (≈70 %) ou 10.1.0.2 (≈30 %) ; si les deux
  sont Unhealthy → bascule p20 → **10.2.0.1** ;
- client **TW** → 10.3.0.1 (≈80 %) ou 10.4.0.1 (≈20 %) ;
- client **FR** (250) → aucune route FR ni EU dans l'ensemble →
  WORLD → **10.0.0.1** ;
- tout Unhealthy → aucune décision geo → NODATA (NOERROR, 0
  réponses) ;
- question **A** sur une route IPv6 uniquement → réponse vide
  (jamais d'IPv6 dans A) ;
- domaine sans règle applicable → même question, comportement
  classique (NODATA : le GEOROUTE n'a pas de rdata A wire).

### Options CLI (`scone relay`)

| Option | Défaut | Rôle |
|---|---|---|
| (aucune — GeoDNS toujours actif, table embarquée) | on | Routage automatique par IP source |
| `--geo-ip-table <path>` | — | (obsolète — table GeoIP imposée par le build ; accepté, ignoré) |
| `--geo-country <u16>` | — | Pays du client, ISO 3166-1 numérique (840=US…) — statique |
| `--geo-continent <u16>` | — | Continent du client, ONU M49 (150=EU…) ; dérivé du pays si omis — statique |
| `--geo-asn <u32>` | — | AS du client (64500…) — statique, nouveau en v2 |
| `--geo-region <u32>` | — | Subdivision du client (identifiant opaque éditeur) — statique, nouveau en v2 |
| `--health-check` | off | Active les vérifications (bascule de santé) |
| `--no-health-check` | off | **Désactive explicitement** les vérifications — GAGNE sur `--health-check` (voir précédences) |
| `--health-ttl-secs <u64>` | 5 | TTL d'une entrée de santé |
| `--health-port <u16>` | 443 | Port TCP des vérifications (non dérivable d'un enregistrement DNS) |

#### Table GeoIP locale (`--geo-ip-table`)

Fichier TEXTE local de préfixes, lu **une fois** au démarrage
(100 000 règles max, `MAX_GEOIP_RULES`). Une règle par ligne, `#` =
commentaire, lignes vides ignorées :

```text
# prefix/mask        country continent asn     region
10.0.0.0/8           840   3          .       .
2001:db8::/32        250   150        64500   11
192.0.2.0/24         158   9
```

- `prefix/mask` : IPv4 (`len ≤ 32`, converti en forme 16 octets,
  queue à zéro — même convention que le wire) ou IPv6
  (`len ≤ 128`). **Les bits d'hôte doivent être à zéro**
  (`10.1.2.3/8` est refusé) ;
- `country` : ISO 3166-1 **numérique** `u16`, requis ;
- `continent` : ONU M49 `u16`, optionnel (`.` ou omis = absent, le
  continent peut alors se dériver du pays) ;
- `asn` : `u32`, optionnel (`.` = absent) ;
- `region` : `u32` opaque, optionnel (`.` = absent).

Résolution : l'**IP source de la requête DNS** (par client :
UDP `recv_from` / TCP `peer_addr` ; inconnue → IP de référence du
résolveur) est comparée sur 16 octets ; la règle au **plus long
préfixe correspondant** gagne (à égalité, la première dans l'ordre du
fichier) ; aucune correspondance → position vide (World).

**Strictesse** : le fichier est configuré par l'opérateur — une ligne
invalide (préfixe malformé, pays non numérique, continent inconnu,
champ en trop…) est une **erreur de démarrage** portant le **numéro
de la ligne fautive** ; jamais une ligne silencieusement ignorée,
jamais une table partielle.

#### Précédences des options

- **Localisateur** : la table embarquée impose la géolocalisation
  (aucune surcharge opérateur — cf. la table imposée plus haut)
  statiques (`--geo-country`/`--geo-continent`/`--geo-asn`/
  `--geo-region`), qui sont ignorées quand la table est fournie.
  Sans table, les options statiques s'appliquent.
- **Santé** : `--no-health-check` est un **drapeau dédié qui gagne**
  sur `--health-check` — la combinaison `--health-check
  --no-health-check` laisse les vérifications désactivées, sans
  ambiguïté d'ordre de saisie. Sans `--health-check`, les
  vérifications sont désactivées par défaut : **Inconnu =
  disponible**, la réponse geo sort même si la cible est morte. (Un
  `NoopChecker` — toujours Healthy — existe aussi pour monter la
  structure de santé sans écarter aucun point de terminaison :
  tests/paranoïa.)

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


## DoH / DoT (transports chiffrés) — via frontend externe

Le serveur DNS de Scone parle **UDP et TCP en clair** (RFC 1035 /
7766) — c'est le périmètre natif, complet et testé. Les transports
chiffrés vers le *client final* (DoH — DNS-over-HTTPS, RFC 8484 ;
DoT — DNS-over-TLS, RFC 7858) sont **délégués à un frontend
standard** : Scone sert les données, le frontend sert le transport.

Pourquoi ce choix : un serveur DoH natif exigerait une pile HTTPS
complète dans le nœud (certificats, renouvellement, HTTP/2) — une
grande surface opérationnelle pour zéro bénéfice avant qu'il y ait
des utilisateurs finaux côté navigateur. Les frontends existants le
font mieux et depuis plus longtemps ; le nœud reste minimal.

### Exemple : unbound faisant du DoH devant un relais Scone

`/etc/unbound/unbound.conf` :

```
server:
    interface: 127.0.0.1
    port: 5354
    do-ip4: yes
    do-udp: no
    do-tcp: yes

    # DoH (RFC 8484) on 443
    https-port: 443
    tls-service-key: "/etc/unbound/scone.key"
    tls-service-pem: "/etc/unbound/scone.pem"

forward-zone:
    name: "."
    forward-addr: 127.0.0.1@5353   # the Scone relay (--dns 127.0.0.1:5353)
```

Le certificat (`scone.pem`/`scone.key`) s'obtient comme pour tout
service TLS — p. ex. certbot/acme.sh sur le domaine du résolveur.
Les clients DoH pointent alors vers
`https://dns.example.tld/dns-query`.

### Exemple : knot-resolver faisant du DoT

`/etc/knot-resolver/kresd.conf` :

```
net.listen('127.0.0.1', 5353, { kind = 'dns' })
net.listen('0.0.0.0', 853, { kind = 'tls' })
tls.cert_file = '/etc/knot-resolver/scone.pem'
tls.key_file  = '/etc/knot-resolver/scone.key'
```

### Ce que le frontend apporte

- **Le chiffrement du canal client→frontend** (le segment hostile :
  le réseau de l'utilisateur) ; le segment frontend→relais reste
  local (même machine ou réseau de confiance).
- **Un cache supplémentaire côté frontend** (cache unbound/knot
  aussi — inoffensif, les TTL Scone restent la borne supérieure).
- **Un point de terminaison stable** (IPv4/IPv6, domaine) pendant que
  les relais bougent.

### Ce que le frontend ne fait PAS

- Il ne *vérifie* rien : chaque réponse servie par le relais est déjà
  vérifiée contre la chaîne avant d'être mise en cache (voir
  « Chemin de résolution »). Le frontend relaie des réponses déjà
  autoritaires.
- Il ne remplace pas la décentralisation : un frontend DoH est un
  service *opéré* — le choix de l'opérateur du résolveur, pas celui
  du protocole. Chaque relais Scone reste indépendamment utilisable
  en UDP/TCP natif.

### Implémentation native future (documentée, non engagée)

Un transport DoH natif (`--doh ADDR`) reste concevable : le format
wire étant identique (la RFC 8484 réutilise les messages DNS
binaires), ce serait un décodage POST/GET + HTTP/2 derrière un proxy
TLS (caddy/nginx gérant les certificats), le nœud lui-même restant
sans TLS. Aucun travail en cours — la demande viendra avec les
utilisateurs finaux.

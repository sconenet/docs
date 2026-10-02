# Méthodologie d'audit de sécurité — Scone (pré-mainnet)

Document de travail. Cible : confiance mise en production. Références
cross-crate : les règles dures d'AGENTS.md (aucune crypto maison,
décodage strict, hashs recalculés, déterminisme) sont le cahier des
charges de l'audit.

## 0. Périmètre et menaces (avant toute ligne de code relue)

Modèle d'adversaire : un nœud p2p arbitrairement hostile (peut envoyer
n'importe quels octets sur QUIC/UDP), un soumetteur de tx hostile, un
opérateur de relay curieux, un vol de data-dir. EXCLU par conception :
le client final (le DNS est publique par nature), la disponibilité
globale (chaos déjà testé).

Table des actifs, par ordre de criticité :
1. **Ownership** (chaîne d'état : Register/Update/Transfer/Assign) —
   un faux positif = vol de domaine.
2. **Signature/possession de clé** (keystore, Ed25519, dérivation
   OwnerId).
3. **Validation d'entrées non fiables** (décodage wire, limites,
   panics).
4. **DoS ressource** (cache DNS, mempool, DHT, sync).
5. **Déterminisme/consensus** (reorg, GC, résurrection d'état).

## 1. Audit statique automatisé (jour 1 — mécanique)

- `cargo audit` (CVE dépendances : libp2p/quinn/rustls/redb/ed25519).
- `cargo geiger` + revue des `unsafe` (attendu ≈ 0).
- `cargo clippy --workspace --all-targets -- -D warnings` (déjà vert,
  à garder comme barrière CI).
- Fuzzing des frontières de décodage (jour 2-3) :
  `cargo-fuzz` sur `scone_protocol::decode_complete` (Message,
  Transaction, Block, SignedDnsRecord, Checkpoint) — c'est LA surface
  où « jamais de panic sur données non fiables » doit tenir. Budget :
  24 h de fuzzing continu, corpus seedé par les tests existants.
  Critère de sortie : zéro crash, zéro timeout, couverture des
  branches de décodage > 95 %.
- Fuzzing du parseur DNS wire (dns.rs `parse_query`, `extract_answers`)
  et du parseur de fichiers de records (domain.rs).

## 2. Revue manuelle ciblée par composant (jours 4-8)

Ordre par criticité des actifs, 1 composant = 1 checklist :

### 2.1 scone-crypto + dérivation d'identité (jour 4)
- OwnerId = BLAKE3(pubkey) : vérifier la canonicalisation d'octets
  (pas de représentation ambiguë → 2 clés, 1 owner).
- Ed25519 : strict DESERIALISATION (clé non canonique rejetée ?),
  vérification à séparation de contexte (chaque usage a son tag de
  domaine : SCONE-TX-SIGN-V1, SCONE-BLOCK-V2, record DHT — aucun
  cross-protocol replay possible).
- Keystore : passphrase → clé (KDF ? itérations ?), zéroisation,
  permissions fichier, pad oracle sur le ciphertext.

### 2.2 Application des transactions (jour 5) — le cœur
Pour CHAQUE variante de Transaction (10 aujourd'hui) :
- La tx est-elle idempotente au replay (txid unique, nonce/séquence) ?
- Le contrôle d'ownership précède-t-il tout effet ? (signer == owner
  appris on-chain, jamais cru depuis la tx elle-même.)
- Séquence UpdateDomain : exact current+1 (pas ≥) — vérifier le
  refus du gap et le refus du replay.
- Renew : bornes temporelles (≤ 3 ans, > maintenant) — le temps vient
  du BLOC, jamais de l'horloge locale (règle déjà fixée, vérifier
  chaque site d'appel).
- AssignDomain/TransferTld : la pubkey de l'assigné est-elle APPRISE
  (le piège du pool sans pubkey — cf. skill) sur tous les chemins.
- GC/expiration : aucune résurrection possible (le paradoxe
  precheck-OK/lookup-OK/push-échoue) — vérifier gc_expired_journaled.

### 2.3 Couche réseau (jour 6)
- Tout octet distant passe par un décodeur strict borné AVANT
  allocation (MAX_* de limits.rs) — chercher les `Vec::with_capacity(
  len_de_l'entrée)` non plafonnés.
- requery/sync GetBlocks : bornes par pair et par cycle (déjà : 128
  blocs/req) — vérifier l'absence de boucle de re-téléchargement.
- Gossip Record : re-vérification chaîne AVANT cache/gossip (vérifié
  par mes tests record_dht) — vérifier aussi le plafond de fanout.
- DHT kad : un pair hostile peut-il forcer le stockage de N records
  (quota par pair ?) ou l'éviction ciblée des records honnêtes ?

### 2.4 Surface DNS (jour 7)
- Cache : injection impossible (clé (qname,qtype) canonique — pas de
  collision casse/encodage), budget mémoire dur (testé), pas de
  cache poisoning via le chemin fallback (TXID + echo question —
  vérifier que l'upstream ne peut pas être spooofé à distance).
- Réponses : AA honnête, TC honnête, pas d'amplification (ratio
  réponse/requête borné — MAX_INFLIGHT_UDP déjà là, vérifier la
  réplication TCP).

### 2.5 Stockage & genèse (jour 8)
- redb : corruption disque → erreur typée, pas de panic (le test
  diskfail existe côté signer — l'équivalent relay ?).
- Genèse : hash couvre le manifeste entier ; deux genèses différentes
  = refus data-dir (testé) ; vérifier l'impossibilité d'injecter une
  genèse APRÈS coup sur un data-dir syncé.

## 3. Tests d'adversaire (jours 9-12) — écrire ce qui manque

Pour chaque finding de la revue manuelle, un test qui le prouve :
- nœud hostile dans le sim (SimNet avec un nœud qui envoie des
  messages malformés/en désaccord) — étendre SimNet d'un rôle
  "byzantin passif" ;
- mesh réel 30 nœuds avec 3 nœuds hostiles (bytes aléatoires, tx
  invalides en rafale, records DHT contrefaits) — la batterie chaos
  existe, ajouter le rôle attaquant ;
- vérifications croisées : deux implémentations du décodage (le doc
  protocol.md est NORMATIF — un auditeur implémente le wire depuis la
  doc SEULE et compare les octets ; toute divergence = bug doc OU
  code).

## 4. Audit externe (si budget) — après 1-3

Périmètre minimal rentable pour un prestataire : scone-crypto,
scone-protocol (wire), l'application des tx (state.rs), ~8-10 k LOC.
Format : 2 semaines, 1 auditeur senior crypto + 1 reviewer. Livrable :
rapport avec sévérité, PoC, correctifs revus. Compter sur la
documentation normative existante (docs/technical) comme base du
cahier des charges — c'est un avantage coût réel.

## 5. Critères de sortie (go/no-go mainnet)

- [ ] cargo-fuzz 24 h sans crash sur les 5 décodeurs
- [ ] Revue 2.2 complète : chaque variante de tx a sa fiche
      (ownership/replay/temps/effets) signée par le relecteur
- [ ] Mesh 30 nœuds dont hostiles : 48 h sans divergence de tip ni
      corruption de cache
- [ ] Divergence doc/code : zéro (vérifié par la double
      implémentation wire)
- [ ] CVE bloquantes : zéro (cargo audit)
- [ ] Audit externe du périmètre crypto/wire/tx fait (si budgété)

## Ordre d'exécution recommandé

1→2.1→2.2 (le cœur, 60 % du risque)→2.3→2.4→2.5→3 en parallèle de
chaque revue (le test d'adversaire s'écrit le jour de la finding,
pas après).

# Méthodologie d'audit de sécurité — Scone (pré-mainnet)

Document de travail. Objectif : confiance pour un déploiement en
production. Références inter-crates : les règles dures de AGENTS.md (pas
de crypto maison, décodage strict, hachages recalculés, déterminisme)
sont la spécification des exigences de l'audit.

## 0. Périmètre et menaces (avant de relire une seule ligne de code)

Modèle d'adversaire : un nœud p2p arbitrairement hostile (peut envoyer
n'importe quels octets via QUIC/UDP), un soumetteur de tx hostile, un
opérateur de relais curieux, un data-dir volé. EXCLUS par conception : le
client final (le DNS est public par nature), la disponibilité globale
(le chaos est déjà testé).

Table des actifs, par ordre de criticité :
1. **La propriété** (chaîne d'état : Register/Update/Transfer/Assign) —
   un faux positif = vol de domaine.
2. **Signature/possession de clé** (coffre de clés, Ed25519, dérivation
   `OwnerId`).
3. **Validation des entrées non fiables** (décodage wire, limites,
   panics).
4. **DoS ressources** (cache DNS, mempool, DHT, sync).
5. **Déterminisme/consensus** (réorg, GC, résurrection d'état).

## 1. Audit statique automatisé (jour 1 — mécanique)

- `cargo audit` (CVEs de dépendances : libp2p/quinn/rustls/redb/ed25519).
- `cargo geiger` + revue des `unsafe` (attendu ≈ 0).
- `cargo clippy --workspace --all-targets -- -D warnings`
  (déjà vert, garder comme barrière CI).
- Fuzzing des frontières de décodage (jours 2-3) :
  `cargo-fuzz` sur `scone_protocol::decode_complete` (Message,
  Transaction, Block, SignedDnsRecord, Checkpoint) — c'est LA
  surface où « jamais de panic sur données non fiables » doit tenir.
  Budget : 24 h de fuzzing continu, corpus amorcé depuis les tests
  existants. Critère de sortie : zéro crash, zéro timeout, couverture
  des branches de décodage > 95 %.
- Fuzzing de l'analyseur wire DNS (dns.rs `parse_query`,
  `extract_answers`) et de l'analyseur de fichier d'enregistrements
  (domain.rs).

## 2. Revue manuelle ciblée par composant (jours 4-8)

Ordonnée par criticité des actifs, 1 composant = 1 checklist :

### 2.1 scone-crypto + dérivation d'identité (jour 4)
- OwnerId = BLAKE3(pubkey) : vérifier la canonicalisation des octets
  (pas de représentation ambiguë → 2 clés, 1 propriétaire).
- Ed25519 : DÉSÉRIALISATION stricte (clé non canonique rejetée ?),
  vérification avec séparation de contexte (chaque usage a son tag de
  domaine : SCONE-TX-SIG-V1, SCONE-BLOCK-V1, enregistrement DHT — pas
  de replay inter-protocole possible).
- Coffre de clés : phrase de passe → clé (KDF ? itérations ?),
  mise à zéro, permissions de fichier, oracle de padding sur le
  chiffré.

### 2.2 Application des transactions (jour 5) — le cœur
Pour CHAQUE variante de Transaction (13 aujourd'hui) :
- La tx est-elle idempotente au replay (txid unique, nonce/séquence) ?
- Le contrôle de propriété précède-t-il tout effet ? (signataire ==
  propriétaire appris on-chain, jamais cru depuis la tx elle-même.)
- Séquence UpdateDomain : exactement courant+1 (pas ≥) — vérifier le
  rejet des trous et le rejet des replays.
- Renew : bornes temporelles (≤ 3 ans, > now) — le temps vient du
  BLOC, jamais de l'horloge locale (règle déjà posée, vérifier chaque
  site d'appel).
- AssignDomain/TransferTld : la pubkey du destinataire est-elle APPRISE
  (le piège du pool sans pubkey — cf. skill) sur tous les chemins.
- GC/expiration : aucune résurrection possible (le paradoxe
  precheck-OK/lookup-OK/push-échoue) — vérifier
  gc_expired_journaled.

### 2.3 Couche réseau (jour 6)
- Chaque octet distant passe par un décodeur strict borné AVANT
  allocation (MAX_* de limits.rs) — chercher les
  `Vec::with_capacity(input_len)` non bornés.
- requery/sync GetBlocks : bornes par pair et par cycle (déjà :
  128 blocs/req) — vérifier l'absence de boucles de
  re-téléchargement.
- Gossip Record : re-vérification contre la chaîne AVANT cache/gossip
  (vérifié par mes tests record_dht) — vérifier aussi le cap de
  fanout.
- kad DHT : un pair hostile peut-il forcer le stockage de N
  enregistrements (quota par pair ?) ou l'éviction ciblée
  d'enregistrements honnêtes ?

### 2.4 Surface DNS (jour 7)
- Cache : injection impossible (clé canonique (qname,qtype) — pas de
  collision de casse/encodage), budget mémoire dur (testé), pas
  d'empoisonnement de cache via le chemin de repli (TXID + echo de
  question — vérifier que l'amont ne peut pas être usurpé à distance).
- Réponses : AA honnête, TC honnête, pas d'amplification (ratio
  réponse/requête borné — MAX_INFLIGHT_UDP déjà là, vérifier la
  réplication TCP).

### 2.5 Stockage & genèse (jour 8)
- redb : corruption disque → erreur typée, pas de panic (le test
  diskfail existe côté signataire — l'équivalent relais ?).
- Genèse : le hachage couvre tout le manifeste ; deux genèses
  différentes = refus de data-dir (testé) ; vérifier l'impossibilité
  d'injecter une genèse APRÈS coup sur un data-dir synchronisé.

## 3. Tests adverses (jours 9-12) — écrire ce qui manque

Pour chaque constat de la revue manuelle, un test qui le prouve :
- nœud hostile dans la sim (SimNet avec un nœud envoyant des messages
  malformés/dissidents) — étendre SimNet avec un rôle « byzantin
  passif » ;
- vrai maillage de 30 nœuds avec 3 nœuds hostiles (octets aléatoires,
  rafales de tx invalides, enregistrements DHT forgés) — la batterie
  chaos existe, ajouter le rôle attaquant ;
- contre-vérifications : deux implémentations du décodage (la doc
  protocol.md est NORMATIVE — un auditeur implémente le wire depuis la
  doc SEULE et compare les octets ; toute divergence = bug de la doc
  OU du code).

## 4. Audit externe (si budget) — après 1-3

Périmètre minimal rentable pour un prestataire : scone-crypto,
scone-protocol (wire), application des tx (state.rs), ~8-10 k LOC.
Format : 2 semaines, 1 auditeur crypto senior + 1 relecteur. Livrable :
rapport avec sévérité, PoCs, correctifs revus. S'appuyer sur la
documentation normative existante (docs/technical) comme base de la
spécification des exigences — un vrai gain de coût.

## 5. Critères de sortie (go/no-go mainnet)

- [ ] cargo-fuzz 24 h sans crash sur les 5 décodeurs
- [ ] Revue 2.2 complète : chaque variante de tx a sa fiche
      (propriété/replay/temps/effets) signée par le relecteur
- [ ] Maillage 30 nœuds incluant des hostiles : 48 h sans divergence
      de sommet ni corruption de cache
- [ ] Divergence doc/code : zéro (vérifiée par la double
      implémentation wire)
- [ ] CVEs bloquantes : zéro (cargo audit)
- [ ] Audit externe du périmètre crypto/wire/tx fait (si
      budgété)

## Ordre d'exécution recommandé

1→2.1→2.2 (le cœur, 60 % du risque)→2.3→2.4→2.5→3 en parallèle de
chaque revue (le test adverse est écrit le jour du constat, pas
après).

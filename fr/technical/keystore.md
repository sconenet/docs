# Coffre de clés — spécification du format de fichier de clés `.sconekey`

> Document normatif. Toute divergence entre ce document et
> `crates/scone-keystore/src/lib.rs` est un bug (identifier la
> source de vérité, la corriger). Référence d'implémentation :
> crate `scone-keystore` (v2).

## Objet

Stocker sur disque une clé privée Ed25519 (la *seed*, 32 octets),
chiffrée par une phrase de passe. La clé ne touche **jamais** le
disque en clair ; tous les secrets (phrase de passe, matériau
dérivé) sont zéroisés en mémoire après usage.

## Vue d'ensemble

- **KDF** : Argon2id (v0x13), paramètres par défaut `m_cost` =
  64 Mio, `t_cost` = 3, `p_cost` = 1 (défauts de la crate
  `argon2`).
- **Chiffrement** : XChaCha20-Poly1305 (AEAD), clé de 32 octets
  dérivée de la KDF, nonce aléatoire de 24 octets par fichier.
- **Contenu chiffré** : exactement la seed Ed25519 (32 octets) ;
  l'étiquette Poly1305 (16 octets) est stockée en suffixe du
  texte chiffré (sortie standard de l'API `encrypt` de l'AEAD).

## Scellement en octets (module `bytes`, web)

`scone_keystore::bytes::{seal_to_bytes, open_from_bytes}`
produit et consomme le MÊME format sans toucher au système de
fichiers : le client web génère l'identité dans le navigateur
(`scone-crypto` + bip39) et scelle les octets exacts du fichier
de clés (téléchargeable, ouvrable par la CLI tel quel — test
d'interopérabilité dans la suite). Le chemin natif
(`create`/`create_with_phrase`) délègue au même scellement : un
unique encodage du format, trois destinations.

## Format binaire (v2) — en-tête de 52 octets + métadonnées publiques + texte chiffré

```text
Offset  Size        Field
0       9           magic: "SCONEKEY1" (ASCII — unchanged since v1)
9       1           version: u8 = 2
10      1           m_cost_mib: u8 (m_cost in MiB; m_cost_kib = value × 1024)
11      1           t_cost: u8 (Argon2id iterations)
12      16          Argon2id salt (random per file)
28      24          XChaCha20-Poly1305 nonce (random per file)
52      32          public_key: Ed25519 public key (32 bytes, IN THE CLEAR)
84      8           created_at: u64 big-endian (Unix, seconds)
92      1           label_len: u8 (0..=255)
93      label_len   label: UTF-8 (readable without the passphrase, may be empty)
—       48          ciphertext (32 bytes encrypted seed) || Poly1305 tag (16 bytes)
```

Les métadonnées `public_key`/`created_at`/`label` sont
publiques : lisibles **sans la phrase de passe**
(`public_metadata_from_bytes`). La clé publique est publique par
nature ; l'étiquette n'est qu'un nom d'affichage — la cacher
n'apporte rien et coûte le listing sans phrase de passe. La
longueur totale dépend de l'étiquette : 141 octets avec une
étiquette vide (KEYFILE_LEN + 32 + 8 + 1), plus la longueur de
l'étiquette.

**v1 (remplacée, exactement 100 octets)** : magic ‖ version=1 ‖
m ‖ t ‖ sel ‖ nonce ‖ texte chiffré — la clé publique n'était
PAS stockée (un fichier verrouillé était anonyme). Les fichiers
v1 restent **lisibles** (métadonnées `None`, phrase de passe
exigée pour `identity show` ; un re-scellage via
`scone identity restore` les migre en v2).

### Choix de conception

- **Paramètres KDF dans le fichier** : `m_cost_mib` et `t_cost`
  sont lus depuis l'en-tête (pas codés en dur à la lecture), ce
  qui permet de les relever dans de futures versions sans casser
  les fichiers existants.
- **Bornes de sécurité à la lecture** : un fichier déclarant
  `m_cost_mib` > 64 ou `t_cost` = 0 ou > 10 est rejeté
  (`Error::InvalidParams`) *avant* toute allocation — un
  fichier malveillant ne peut pas forcer un nœud à allouer des
  gigaoctets ou boucler indéfiniment. Le défaut à l'écriture
  est `m_cost_mib` = 64, `t_cost` = 3.
- **Magic + version explicites** : un fichier qui n'est pas un
  fichier de clés (`Error::InvalidFile`) ou d'une version non
  prise en charge (`Error::UnsupportedVersion`) est rejeté
  proprement, sans panic.
- **Phrase de passe erronée** : l'échec d'authentification
  Poly1305 n'indique pas *pourquoi* il échoue ; une phrase de
  passe erronée et un texte chiffré corrompu sont donc
  indiscernables et remontent comme la même erreur distincte
  `Error::WrongPassphrase`.

## Dérivation de la clé de chiffrement

```text
key = Argon2id(passphrase, salt, m_cost = m_cost_mib × 1024 KiB,
               t_cost = t_cost, p_cost = 1, output = 32 bytes)
```

Puis :

```text
ciphertext || tag = XChaCha20-Poly1305-Seal(key, nonce, plaintext = seed)
```

Pas de données associées (AAD) en v1. Le nonce de 24 octets,
aléatoire par fichier, rend le risque de réutilisation de nonce
négligeable (les fichiers de clés ne sont écrits qu'une fois).

## API (`scone-keystore`)

| Fonction | Rôle |
|---|---|
| `create(path, passphrase) -> GeneratedKey` | Génère une clé, écrit le fichier de clés chiffré (crée les répertoires parents) ; **refuse d'écraser** un fichier existant (`Error::KeyfileExists`) ; retourne la clé en mémoire |
| `create_overwriting(path, passphrase) -> GeneratedKey` | Pareil, mais remplace explicitement un fichier de clés existant |
| `open(path, passphrase) -> SigningKey` | Déchiffre et retourne la clé de signature |
| `list(dir) -> Vec<KeyEntry>` | Liste les fichiers `*.sconekey` d'un répertoire (triés par nom) ; répertoire absent = liste vide |

`GeneratedKey.signing_key` est zéroisée au drop par
`ed25519-dalek` (fonctionnalité `zeroize` par défaut). La clé
AEAD dérivée, la seed et le clair déchiffré sont enveloppés dans
`zeroize::Zeroizing` : le Drop couvre tous les chemins de sortie,
y compris les erreurs propagées par `?`.

## Permissions des fichiers

Sur Unix, le fichier de clés est créé avec les permissions
`0600` (lecture/écriture par le propriétaire seul), via
`OpenOptions` + `set_permissions` — indépendamment de l'umask
du processus. L'existence et la création sont atomiques
(`create_new`, `O_EXCL`) : deux générations concurrentes ne
peuvent pas s'écraser mutuellement en silence.

## CLI

```bash
scone identity generate <n> [--dir <directory>] [--passphrase-env VAR] [--force]
scone identity list [--dir <directory>]
scone identity show <n> [--dir <directory>] [--passphrase-env VAR]
```

- Répertoire par défaut : `$HOME/.scone/keys`.
- Le nom d'identité est strictement validé : vide, `.`, `..`,
  `/`, `\\\\`, NUL, point initial et suffixe `.sconekey` sont
  rejetés (le nom devient un nom de fichier dans le répertoire
  des clés — aucun composant de chemin ne peut s'échapper du
  coffre de clés).
- Sans `--passphrase-env`, la phrase de passe est demandée sur
  le terminal **avec saisie masquée** (`rpassword` : pas d'écho
  ; deux fois avec confirmation pour `generate`).
- `identity generate` **affiche la phrase de récupération BIP39
  (24 mots)** exactement une fois — entropie fraîche de 256
  bits, les mots seuls régénèrent la même clé (`scone identity
  restore`) ; ils ne sont jamais stockés nulle part.
- `identity generate` **refuse d'écraser** un fichier de clés
  existant ; `--force` le remplace explicitement (nouveau
  sel/nonce, nouvelle clé, nouvelle phrase).
- `identity list` affiche, pour les fichiers v2, la clé publique
  et l'étiquette **sans la phrase de passe** (métadonnées
  publiques).
- `identity show` lit les métadonnées publiques du fichier v2
  et affiche la clé publique, l'`OwnerId` en hexadécimal
  minuscule (64 caractères, même convention que `DomainId`),
  l'étiquette et la date de création **sans la phrase de
  passe** ; un fichier v1 (sans métadonnées) la demande
  toujours.

## Ce que ce format ne fait volontairement pas

- Pas de clés multiples par fichier (une seed par fichier).
- Pas de changement de phrase de passe (si un jour cela vient :
  réécrire le fichier avec un nouveau sel/nonce).


## Identité par défaut (cli-simplify-v1)

Les commandes qui ont besoin d'une identité de signature
(`domain register`, `domain set`, `domain update`,
`domain renew`) utilisent l'identité UNIQUE du coffre de clés
sans `--identity` quand il n'y en a exactement qu'une. Avec
plusieurs identités, la commande échoue et les liste
(`scone identity list`) ; passez `--identity <nom>`
explicitement.

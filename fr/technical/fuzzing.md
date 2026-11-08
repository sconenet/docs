# Fuzzing — robustesse aux entrées hostiles

Le réseau délivre des octets hostiles : n'importe quel pair peut
envoyer n'importe quoi via QUIC/UDP, un disque peut contenir des
fichiers corrompus, un keyfile peut être tronqué en cours
d'écriture. Chaque décodeur d'entrée non fiable doit satisfaire un
invariant (SECURITY_AUDIT §10, méthodologie d'audit AM2) :

- **ne jamais paniquer**,
- **ne jamais allouer une mémoire non bornée**,
- **ne jamais boucler à l'infini**.

Un rejet typé (`Err`) est le résultat attendu — c'est aussi le
proxy de couverture rapporté par le harnais. L'infrastructure de
fuzzing asserte cet invariant en continu au lieu de se fier à la
seule relecture de code.

## Les deux couches

| | `fuzz-smoke/` | `fuzz/` |
|---|---|---|
| Outil | runner de mutation déterministe, std-only (maison) | cargo-fuzz, libFuzzer + AddressSanitizer |
| Toolchain | stable — ne bloque jamais sur nightly | nightly + g++ requis |
| Workspace | membre (`cargo test -p fuzz-smoke`) | **hors** workspace par conception |
| Rôle | smoke CI + longs runs opérateur | campagnes profondes guidées par la couverture |

### Couche 1 — `fuzz-smoke/` (déterministe)

Chaque cible est une simple `fn(data: &[u8])` qui appelle le VRAI
décodeur de production, enveloppée dans `catch_unwind`. Toute
panique = bug de décodeur : l'entrée fautive est sauvegardée sous
`fuzz-smoke/artifacts/<target>/` et le run sort non-zéro. Le PRNG
est un SplitMix64 amorcé : **la même seed rejoue exactement la
même séquence d'entrées** — un échec est toujours reproductible.

`cargo test --workspace` exécute la batterie smoke : 10 000
itérations déterministes par vraie cible (seeds fixes par cible),
le scénario de panique plantée (capture + sauvegarde d'artifact +
poursuite) et le contrat de déterminisme (même seed → même
séquence). Les LONGS runs (≥ 50k, p. ex. 1M itérations/cible)
restent l'étape manuelle de l'opérateur avant mainnet.

### Couche 2 — `fuzz/` (guidée par la couverture)

La migration cargo-fuzz est délibérément mécanique : chaque cible
est déjà un `LLVMFuzzerTestOneInput` prêt à l'emploi, et les
corpus embarqués sont réutilisables comme seed dirs libFuzzer. Elle
vit hors du workspace pour que les exigences nightly/ASan ne
fuient jamais dans la CI. Budget d'audit : 24 h de fuzzing
continu ; critère de sortie — zéro crash, zéro timeout, couverture
des branches des décodeurs > 95 %.

## Cibles

| Nom (`--target`) | Surface de production testée |
|---|---|
| `wire` | `decode_complete::<Message>` — le décodeur de trames wire |
| `dns_query` | `parse_query` — le parseur de requêtes DNS UDP/TCP |
| `dns_answers` | `extract_answers` — l'extraction des réponses DNS |
| `record_file` | `scone::record_file` — le parseur de record-file du CLI |
| `keyfile` | parse/open des octets `.sconekey` |
| `panic_scratch` | canari du harnais — panique exprès pour prouver que le chemin de capture d'artifact fonctionne |

`--target all` couvre les cinq vrais décodeurs (le canari est
opt-in).

## Lancer

```bash
# lister les cibles
cargo run -p fuzz-smoke -- --list

# run court (build debug, 10k itérations par défaut)
cargo run -p fuzz-smoke -- --target wire

# les cinq décodeurs d'un coup (release)
cargo run -p fuzz-smoke --release -- --target all --iters 50000

# le run opérateur pré-mainnet — une cible, long
cargo run -p fuzz-smoke --release -- --target wire --iters 1000000

# seed fixe + emplacement d'artifacts personnalisé
cargo run -p fuzz-smoke --release -- --target wire --seed 0xC0FFEE \
  --artifacts-dir /tmp/fuzz-out
```

## À quoi ressemble un échec

Un crash sauvegarde les octets fautifs exacts dans le répertoire
d'artifacts (par défaut `fuzz-smoke/artifacts/<target>/`) et le
runner sort non-zéro. Reproduire depuis l'artifact (entrée de
corpus) ou depuis la seed. Tout le reste — les rejets typés
`Err(...)` — est le décodeur qui fait son travail et est rapporté
comme couverture, jamais comme échec.

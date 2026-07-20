# Couverture — workflow indépendant (Rust)

```yaml
name: Code Coverage
```

← [Pipelines](../../pipelines/pipelines.fr.md) | ← [Pipeline needs: — chaîne courte](../../pipelines/needs/pipeline.fr.md#implémentation-b--chaîne-courte-trois-étapes)

> **Modèle prêt à copier** :
> [`shared/github-ci/coverage/rust-coverage.yml`](../../../shared/github-ci/coverage/rust-coverage.yml).
> Remplacer `<feature_name>` par la ou les fonctionnalités Cargo réelles du crate, ou supprimer les steps
> correspondants si le crate n'a pas de fonctionnalités optionnelles.

## Vue d'ensemble

Ce pipeline GitHub Actions mesure la couverture de code Rust avec `cargo-llvm-cov` et télécharge
automatiquement le rapport vers Codecov. Il documente une **troisième variante d'architecture**, distincte
des deux modèles principaux du dépôt :

- ce n'est pas `workflow_run` : aucun fichier n'attend l'achèvement d'un autre ;
- ce n'est pas `needs:` non plus : il n'y a qu'un seul job dans ce fichier, rien à chaîner en interne ;
- c'est un **second point d'entrée indépendant**, qui vit à côté du pipeline principal (`ci.yml`,
  [chaîne courte du modèle needs:](../../pipelines/needs/pipeline.fr.md#implémentation-b--chaîne-courte-trois-étapes))
  et se déclenche sur son propre événement Git, sans lien de dépendance avec lui.

Voir [Deux architectures de pipeline](../../architecture/ci-models.fr.md) pour le comparatif complet des
deux modèles principaux ; cette page-ci couvre le cas où un projet a besoin d'un troisième fichier qui ne
rentre dans aucun des deux.

## Déclenchement du pipeline

```yaml
on:
  push:
    branches:
      - master
      - "feature/**"
  pull_request:
    branches:
      - master

env:
  CARGO_TERM_COLOR: always
  RUST_BACKTRACE: 1
```

### Conditions de déclenchement

1. **Push sur `master`** : mesure la couverture de la branche de référence
2. **Push sur toute branche `feature/**`** : donne un retour de couverture dès le développement d'une
   fonctionnalité, sans attendre une fusion
3. **Pull request vers `master`** : vérifie la couverture avant fusion

**Ce qui distingue ce déclenchement de l'implémentation A du modèle `needs:`** (couverture chaînée,
restreinte aux tags) : ici la couverture tourne à **chaque push sur une branche de travail**, pas seulement
à la release. C'est le choix pertinent quand l'équipe veut un retour de couverture continu pendant le
développement, indépendamment du cycle de tag — au prix de runs de CI plus fréquents.

`CARGO_TERM_COLOR` et `RUST_BACKTRACE` ne conditionnent pas le déclenchement : ce sont des variables
d'environnement qui améliorent la lisibilité des logs (sortie colorée, trace complète en cas de panique),
définies une fois au niveau du workflow plutôt que répétées à chaque step.

## Architecture du pipeline

```yaml
jobs:
  coverage:
    name: Generate Coverage Report
    runs-on: ubuntu-latest
```

Un seul job — contrairement aux pipelines Python de ce dépôt, il n'y a rien à chaîner ici : toute la
mesure de couverture (installation, exécution, upload) tient dans une séquence de steps sur le même
runner.

## Étapes détaillées du pipeline

### Étape 1 : Récupération du code source

```yaml
- name: Checkout repository
  uses: actions/checkout@v4
```

Rien de particulier ici : ce pipeline se déclenche directement sur l'événement Git (`push`/`pull_request`),
il récupère donc naturellement le bon commit — pas de `head_sha` à gérer, contrairement aux pipelines
déclenchés par `workflow_run`.

### Étape 2 : Installation de la chaîne d'outils Rust

```yaml
- name: Install Rust toolchain
  uses: dtolnay/rust-toolchain@stable
  with:
    components: llvm-tools-preview
```

Le composant `llvm-tools-preview` est requis par `cargo-llvm-cov` : c'est lui qui fournit l'instrumentation
LLVM utilisée pour mesurer la couverture ligne par ligne. Sans ce composant, l'étape 4 échouerait.

### Étape 3 : Mise en cache

```yaml
- name: Cache cargo registry
  uses: actions/cache@v4
  with:
    path: ~/.cargo/registry
    key: ${{ runner.os }}-cargo-registry-${{ hashFiles('**/Cargo.lock') }}
    restore-keys: |
      ${{ runner.os }}-cargo-registry-

- name: Cache cargo index
  uses: actions/cache@v4
  with:
    path: ~/.cargo/git
    key: ${{ runner.os }}-cargo-index-${{ hashFiles('**/Cargo.lock') }}
    restore-keys: |
      ${{ runner.os }}-cargo-index-

- name: Cache cargo build
  uses: actions/cache@v4
  with:
    path: target
    key: ${{ runner.os }}-cargo-build-${{ hashFiles('**/Cargo.lock') }}
    restore-keys: |
      ${{ runner.os }}-cargo-build-
```

Trois caches distincts plutôt qu'un seul : le registre des crates téléchargées (`~/.cargo/registry`), l'index
git des dépendances (`~/.cargo/git`), et les artefacts de compilation déjà construits (`target/`). Les
séparer permet à GitHub Actions d'invalider chacun indépendamment — un changement de `Cargo.lock` invalide
les trois (même clé de hash), mais si un seul cache manque au premier run, les deux autres restent
exploitables. Cette étape n'existe pas côté Python : `pip`/`uv` n'ont pas d'équivalent direct à la
compilation incrémentale de `target/`, qui est ce qui coûte le plus cher à reconstruire ici.

### Étape 4 : Installation de cargo-llvm-cov

```yaml
- name: Install cargo-llvm-cov
  uses: taiki-e/install-action@cargo-llvm-cov
```

Installe le binaire `cargo-llvm-cov` via une action dédiée (binaire précompilé) plutôt que par
`cargo install` — plus rapide, pas de compilation depuis les sources à chaque run.

### Étape 5 : Génération de la couverture, une fois par combinaison de fonctionnalités

```yaml
- name: Generate code coverage — default features
  run: |
    cargo llvm-cov --all-targets --workspace \
      --lcov --output-path lcov-default.info

- name: Generate code coverage — feature wasm-plugins
  run: |
    cargo llvm-cov --all-targets --features wasm-plugins --workspace \
      --lcov --output-path lcov-wasm-plugins.info
```

Deux runs distincts plutôt qu'un seul avec `--all-features` : chaque run correspond à une combinaison de
fonctionnalités (`features` Cargo) réellement déclarée et utilisée dans le projet. Un unique run
« toutes fonctionnalités activées » masquerait le code mort si deux fonctionnalités s'excluent
mutuellement en pratique, et ne refléterait pas les configurations que les utilisateurs du crate activent
réellement. Chaque run produit son propre fichier `.info`, fusionnés à l'étape suivante par l'action
Codecov elle-même (`files:` accepte une liste).

Ajouter une nouvelle combinaison de fonctionnalités à mesurer, c'est ajouter un nouveau step sur le même
modèle — voir le commentaire laissé dans le fichier source pour la prochaine fonctionnalité prévue
(activée seulement quand elle sera réellement déclarée dans `Cargo.toml`, pas en anticipation).

### Étape 6 : Téléchargement vers Codecov

```yaml
- name: Upload coverage to Codecov
  uses: codecov/codecov-action@v4
  with:
    files: lcov-default.info,lcov-wasm-plugins.info
    fail_ci_if_error: false
    flags: unittests
    name: codecov-umbrella
    token: ${{ secrets.CODECOV_TOKEN }}
    verbose: true
```

`files:` liste les deux rapports générés à l'étape précédente — Codecov les agrège en un seul rapport
umbrella (`name: codecov-umbrella`) plutôt que de les traiter comme deux mesures séparées.

`fail_ci_if_error: false` — différence notable avec le pipeline Python de couverture (implémentation A du
modèle `needs:`, `fail_ci_if_error: true`) : là-bas, ce job est sur le chemin critique d'une release taguée,
un échec d'upload doit donc bloquer. Ici, la couverture tourne à chaque push sur une branche de travail,
hors de tout cycle de release — un échec ponctuel de Codecov (service tiers indisponible, par exemple) ne
doit pas faire échouer le pipeline pour autant.

### Étape 7 : Archivage des rapports en artefact

```yaml
- name: Archive code coverage results
  uses: actions/upload-artifact@v4
  with:
    name: code-coverage-report
    path: |
      lcov-default.info
      lcov-wasm-plugins.info
    retention-days: 30
```

Conserve les fichiers `.info` bruts comme artefact du run, indépendamment de l'upload vers Codecov —
permet de retélécharger un rapport précis a posteriori (debug d'un écart de couverture, audit) sans avoir à
relancer le pipeline. Rétention à 30 jours, cohérent avec les autres artefacts de qualité documentés dans
ce dépôt.

### Étape 8 : Résumé affiché dans les logs

```yaml
- name: Display coverage summary
  run: |
    echo "📊 Coverage Report Generated"
    echo "Branch: ${{ github.ref_name }}"
    echo "View detailed report at: https://codecov.io/gh/${{ github.repository }}"
```

Purement informatif — facilite la lecture rapide du run dans l'onglet Actions sans avoir à ouvrir Codecov
pour retrouver le lien du rapport.

## Flux de travail complet

```text
1. Push sur master, sur une branche feature/**, ou pull request vers master
   ↓
2. Installation toolchain + cache
   ↓
3. Génération de la couverture (une fois par combinaison de fonctionnalités)
   ↓
4. Téléchargement vers Codecov (agrégation des rapports)
   ↓
5. Archivage des rapports bruts en artefact
```

Aucune dépendance vers ou depuis le pipeline `ci.yml` (fmt/clippy/test/doc, voir
[implémentation B du modèle needs:](../../pipelines/needs/pipeline.fr.md#implémentation-b--chaîne-courte-trois-étapes)) :
les deux fichiers tournent en parallèle sur le même push, chacun avec son propre badge de statut.

## Résultat

- **Succès** ✅ : la couverture est générée pour chaque combinaison de fonctionnalités et le rapport est
  téléchargé vers Codecov
- **Échec** ❌ : uniquement si la génération de couverture elle-même échoue (`cargo llvm-cov`) — un échec
  d'upload Codecov, lui, n'échoue pas le pipeline (voir étape 6)

## Voir aussi

- [Pipeline — chaînage de dépendances mono-fichier](../../pipelines/needs/pipeline.fr.md) — le pipeline
  principal avec lequel ce workflow de couverture cohabite
- [Deux architectures de pipeline](../../architecture/ci-models.fr.md)
- [Pipeline de Couverture de Code Python](../python/coverage.fr.md) — l'équivalent côté modèle
  `workflow_run`, chaîné et restreint aux tags
- [Coverage — English version](coverage.en.md)

---

## Licence

[CC BY-NC 4.0](../../../LICENSE.txt) — documentation et fichiers de configuration.

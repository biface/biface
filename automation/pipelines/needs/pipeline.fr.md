# Pipeline — chaînage de dépendances mono-fichier (`needs:`)

← [Pipelines](../pipelines.fr.md) | _English version → [pipeline.en.md](pipeline.en.md)_

## Vue d'ensemble

Ce modèle regroupe toutes les étapes du pipeline dans un **unique fichier** de workflow. Chaque job déclare
explicitement de quel(s) job(s) il dépend via `needs:` ; GitHub Actions construit le graphe d'exécution à
partir de ces déclarations. Le principe général (avantages, limites, comparatif avec `workflow_run`) est
expliqué dans
[Deux architectures de pipeline](../../architecture/ci-models.fr.md#modèle-2--chaînage-de-dépendances-needs).

Deux implémentations réelles sont documentées ci-dessous, côte à côte : une **chaîne longue** (six étapes,
Python) et une **chaîne courte** (trois étapes, Rust). La différence entre les deux n'est **pas** le
langage — c'est le nombre d'étapes que le projet a besoin de chaîner ; on aurait tout aussi bien pu avoir
une chaîne longue en Rust ou une chaîne courte en Python. Les fichiers complets sont dans
`shared/github-ci/needs/` ; les extraits ci-dessous ne montrent que ce qui illustre le modèle.

## Squelette générique

```yaml
name: <nom du pipeline>

on:
  push:
    branches:
      - "**"
    tags:
      - "v[0-9]+.[0-9]+.[0-9]+"
      - "v[0-9]+.[0-9]+.[0-9]+rc[0-9]+"
  pull_request:
    branches:
      - "**"

jobs:
  <job_1>:
    name: <libellé lisible>
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # ... mise en place de l'environnement + exécution de <job_1>

  <job_2>:
    name: <libellé lisible>
    runs-on: ubuntu-latest
    needs: <job_1>
    # if: <condition optionnelle, ex. uniquement sur tag>
    steps:
      - uses: actions/checkout@v4
      # ... mise en place de l'environnement + exécution de <job_2>
```

Tous les jobs partagent le même déclencheur Git (`on:`) — contrairement au modèle `workflow_run`, il n'y a
qu'un seul point d'entrée pour l'ensemble de la chaîne.

## Implémentation A — chaîne longue (six étapes, Python)

> **Modèle prêt à copier** :
> [`shared/github-ci/needs/python-ci.yaml`](../../../shared/github-ci/needs/python-ci.yaml). Remplacer
> `<package_name>` par le nom réel du paquet, ajuster la matrice de versions Python, et adapter ou retirer
> le service conteneurisé d'exemple (`httpbin`) selon les besoins des tests d'intégration.

Séquence : `quality` → `test-unit` → `test-integration` → `coverage` → `build` → `publish-pypi` /
`publish-testpypi`. Les quatre dernières étapes sont restreintes aux tags
(`if: startsWith(github.ref, 'refs/tags/')`) — elles ne consomment des minutes de CI que lors d'une
release.

```yaml
jobs:
  quality:
    name: Quality checks
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Install uv
        uses: astral-sh/setup-uv@v4
      - run: uv pip install --system tox tox-uv
      - run: tox -e ci-quality

  test-unit:
    name: Unit tests (Python ${{ matrix.python-version }})
    runs-on: ubuntu-latest
    needs: quality
    strategy:
      fail-fast: false
      matrix:
        python-version: ["3.10", "3.11", "3.12", "3.13"]
        experimental: [false]
        include:
          - python-version: "3.14"
            experimental: true
    continue-on-error: ${{ matrix.experimental }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: |
          PYVER=$(echo "${{ matrix.python-version }}" | tr -d '.')
          tox -e py${PYVER}

  test-integration:
    name: Integration tests
    runs-on: ubuntu-latest
    needs: test-unit
    if: startsWith(github.ref, 'refs/tags/')
    # Service conteneurisé — httpbin (ghcr.io/mccutchen/go-httpbin) sert ici
    # d'exemple concret de dépendance externe. Remplacer <service-name>/
    # <service-image>/<port> par la dépendance réelle des tests d'intégration,
    # ou retirer ce bloc si aucune n'est nécessaire.
    services:
      <service-name>:            # ex. httpbin
        image: <service-image>   # ex. ghcr.io/mccutchen/go-httpbin
        ports:
          - <port>:<port>        # ex. 8080:8080
    steps:
      - uses: actions/checkout@v4
      - run: tox -e integration

  coverage:
    name: Coverage
    runs-on: ubuntu-latest
    needs: test-integration
    if: startsWith(github.ref, 'refs/tags/')
    steps:
      - uses: actions/checkout@v4
      - run: tox -e coverage
      - uses: codecov/codecov-action@v5
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/coverage.xml
          fail_ci_if_error: true

  build:
    name: Verify version and build package
    runs-on: ubuntu-latest
    needs: coverage
    if: startsWith(github.ref, 'refs/tags/')
    steps:
      - uses: actions/checkout@v4
      - name: Verify version consistency
        id: version
        run: |
          # Compare le tag Git, pyproject.toml et src/<package_name>/__static__.py
          # — voir le fichier modèle pour le script complet
      - run: python -m build
      - uses: actions/upload-artifact@v4
        with:
          name: python-package-distributions
          path: dist/
    outputs:
      version: ${{ steps.version.outputs.version }}
      tag: ${{ steps.version.outputs.tag }}

  publish-pypi:
    name: Publish to PyPI
    runs-on: ubuntu-latest
    needs: build
    if: startsWith(github.ref, 'refs/tags/') && !contains(github.ref_name, 'rc')
    permissions:
      contents: write
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: python-package-distributions
          path: dist/
      - uses: pypa/gh-action-pypi-publish@release/v1
        with:
          password: ${{ secrets.PYPI_API_TOKEN }}
          skip-existing: true
      - uses: softprops/action-gh-release@v3
        with:
          tag_name: ${{ needs.build.outputs.tag }}
          prerelease: false
          files: dist/*
```

**Remarques sur cette implémentation :**

- `publish-testpypi` (non montré ci-dessus, voir le fichier modèle) est un job séparé, même structure que
  `publish-pypi`, qui publie vers TestPyPI quand le tag contient `rc` (`if: ... &&
  contains(github.ref_name, 'rc')`) — les deux jobs `publish-*` dépendent tous deux de `build` et se
  distinguent uniquement par leur condition `if:` ;
- la couverture est **un job de la chaîne** (`needs: test-integration`), pas un workflow séparé ; elle
  n'est calculée que sur tag, sur la suite complète de tests ;
- le job `build` expose ses sorties (`outputs:`) pour que les jobs `publish-*` puissent récupérer le tag
  sans le recalculer ;
- le service `httpbin` est un exemple concret, pas une prescription — voir le commentaire d'adaptation en
  tête du fichier modèle.

## Implémentation B — chaîne courte (trois étapes, Rust)

> **Modèle prêt à copier** :
> [`shared/github-ci/needs/rust-ci.yml`](../../../shared/github-ci/needs/rust-ci.yml). Remplacer
> `<feature_name>` par la ou les fonctionnalités Cargo réelles du crate (ou retirer les steps
> correspondants si le crate n'a pas de fonctionnalités optionnelles), et `<crate_name>` par le crate par
> défaut du workspace pour la redirection de documentation.

Séquence : `fmt` → `clippy` **et** `test` (en parallèle, tous deux dépendant de `fmt`) → `doc` (dépend des
deux précédents, restreint à la branche principale).

```yaml
jobs:
  fmt:
    name: Format
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cargo fmt --check

  clippy:
    name: Clippy
    runs-on: ubuntu-latest
    needs: fmt
    steps:
      - uses: actions/checkout@v4
      - run: cargo clippy -- -D warnings
      - run: cargo clippy --features <feature_name> -- -D warnings

  test:
    name: Test
    runs-on: ubuntu-latest
    needs: fmt
    steps:
      - uses: actions/checkout@v4
      - run: cargo test
      - run: cargo test --features <feature_name>
      - run: cargo test --doc
      - run: cargo test --doc --features <feature_name>

  doc:
    name: Documentation
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/master' && github.event_name == 'push'
    needs: [clippy, test]
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - run: cargo doc --no-deps --document-private-items
      - run: |
          echo '<meta http-equiv="refresh" content="0; url=<crate_name>/index.html">' \
            > target/doc/index.html
      - uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./target/doc
```

**Remarques sur cette implémentation :**

- `clippy` et `test` partagent le même `needs: fmt` — ils s'exécutent **en parallèle** l'un de l'autre,
  chacun attendant seulement `fmt` ; c'est l'exemple concret d'un `needs:` à un seul prédécesseur utilisé
  deux fois en éventail, plutôt qu'une chaîne strictement linéaire ;
- `doc` illustre `needs: [clippy, test]` — une dépendance sur **plusieurs** jobs, ce que le modèle
  `workflow_run` ne sait pas exprimer nativement (il faudrait déclencher `doc` sur l'achèvement de deux
  workflows différents, avec deux vérifications de `conclusion` en parallèle dans le même fichier) ;
- chaque commande (`clippy`, `test`) est répétée une fois par fonctionnalité Cargo déclarée, plutôt qu'un
  unique run `--all-features` — voir la justification détaillée dans
  [Couverture — workflow indépendant](../../coverage/rust/coverage.fr.md#étape-5--génération-de-la-couverture-une-fois-par-combinaison-de-fonctionnalités),
  qui applique le même principe ;
- ici, **la couverture ne fait pas partie de ce fichier** — voir ci-dessous.

## Où va la couverture, dans ce modèle ?

Les deux implémentations ci-dessus illustrent deux façons différentes de placer la couverture, toutes deux
compatibles avec le modèle `needs:` :

- **couverture intégrée à la chaîne** (implémentation A, Python) : un job `coverage`, avec son propre
  `needs:`, restreint aux tags — c'est un maillon de plus dans le même fichier ;
- **couverture en workflow indépendant** (implémentation B, Rust) : un second fichier de workflow
  (`shared/github-ci/coverage/rust-coverage.yml`), qui ne dépend de rien dans `rust-ci.yml` — il se
  déclenche seul, sur ses propres conditions de push. Ce n'est **pas** un cas de `workflow_run` (aucun
  fichier n'attend l'achèvement de l'autre) : les deux fichiers sont simplement indépendants. Voir
  [Couverture — workflow indépendant](../../coverage/rust/coverage.fr.md) pour le détail de cette variante.

Le choix entre les deux dépend surtout de la fréquence à laquelle la couverture doit tourner : si elle ne
doit s'exécuter que sur tag comme le reste de la release, l'intégrer à la chaîne évite un fichier de plus ;
si elle doit tourner à chaque push sur des branches de travail (retour rapide, indépendant du cycle de
release), un workflow séparé est plus approprié. Rien n'empêche non plus l'inverse (couverture chaînée en
Rust, workflow indépendant en Python) — le choix suit le besoin du projet, pas son langage.

---

## Licence

[CC BY-NC 4.0](../../../LICENSE.txt) — documentation et fichiers de configuration.

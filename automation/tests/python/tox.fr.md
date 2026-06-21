# Configuration Tox — Guide de référence

← [Automation](../../automation.md)

## Présentation

**Tox** est un outil d'automatisation qui crée des environnements virtuels isolés pour exécuter des commandes
reproductibles. Associé à **tox-uv**, qui remplace pip par uv pour l'installation des dépendances, il constitue
la base du workflow qualité de référence.

Le fichier `tox.ini` centralise la définition de tous les environnements : tests, vérifications qualité, formatage,
sécurité, workflows complets. Une seule source de vérité, utilisable identiquement en local et en CI/CD.

---

## Architecture `.tox-config/`

La configuration est séparée du fichier `tox.ini` dans un répertoire dédié. Cela permet de modifier les
versions Python testées ou les dépendances sans toucher à `tox.ini`.

```
.tox-config/
├── versions.txt          ← versions Python à tester (une par ligne, # pour commenter)
├── coverage-version.txt  ← version Python pour le rapport de couverture
├── requirements/
│   ├── base.txt          ← pytest, pytest-cov
│   ├── full.txt          ← tous les outils réunis
│   ├── format.txt        ← black, isort
│   ├── linter.txt        ← flake8
│   ├── security.txt      ← bandit
│   └── type-check.txt   ← basedpyright
└── scripts/
    ├── test.sh           ← exécution séquentielle multi-versions
    └── coverage.sh       ← génération du rapport de couverture
```

---

## Section par section

### `[tox]` — Configuration générale

```ini
[tox]
minversion = 4.0
requires = tox-uv
toxworkdir = {toxinidir}/.tox
```

- `minversion = 4.0` : garantit la compatibilité avec les fonctionnalités utilisées
- `requires = tox-uv` : installe le plugin uv automatiquement si absent
- `toxworkdir` : répertoire de travail des environnements virtuels

### `[gh-actions]` — Intégration GitHub Actions

```ini
[gh-actions]
python =
    3.10: py310
    3.11: py311
    3.12: py312
    3.13: py313
    3.14: py314
```

Fait correspondre les versions Python de la matrice GitHub Actions aux environnements tox. Quand GitHub Actions
utilise Python 3.12, tox active automatiquement l'environnement `py312`.

### `[testenv]` — Environnement de test par défaut

```ini
[testenv]
uv_seed = true
setenv =
    PYTHONPATH = {toxinidir}/src:{toxinidir}/tests
deps =
    -r {toxinidir}/.tox-config/requirements/base.txt
commands =
    pytest --import-mode=importlib --cov=<package_name> --cov-report=term-missing
```

- `uv_seed = true` : amorce l'environnement uv avec pip — nécessaire pour les outils qui appellent pip directement
- `PYTHONPATH` : permet aux imports de fonctionner sans manipulation de `sys.path`
- `--import-mode=importlib` : mode d'import moderne, évite une classe de bugs silencieux liés à la résolution des modules

Cet environnement s'applique à tous les environnements Python (`py310`, `py311`, etc.) qui n'ont pas de
configuration spécifique.

---

## Environnements de vérification

### `basedpyright` — Vérification de types

```ini
[testenv:basedpyright]
deps =
    -r {toxinidir}/.tox-config/requirements/type-check.txt
allowlist_externals = bash
commands =
    bash -c "basedpyright src; code=$?; [ $code -le 1 ]"
```

**basedpyright** est un vérificateur de types statique strict, basé sur le moteur Pyright de Microsoft.
Plus strict que mypy, il détecte les incompatibilités de types, les attributs inexistants et les appels incorrects.

Le wrapper bash est nécessaire parce que basedpyright distingue deux niveaux de sortie :
- Code `0` : aucun problème
- Code `1` : avertissements (non bloquants)
- Code `2` : erreurs (bloquantes)

`[ $code -le 1 ]` tolère les avertissements tout en bloquant sur les erreurs.

### `flake8` — Lint

```ini
[testenv:flake8]
commands =
    flake8 src tests
```

Vérifie les conventions de style non couvertes par black : complexité cyclomatique, imports inutilisés,
variables non utilisées. Ne modifie rien.

### `black-check` et `isort-check` — Vérification du formatage

```ini
commands =
    black --check --diff --target-version py310 src tests
    isort --check-only --diff src tests
```

Modes de vérification uniquement (`--check`, `--check-only`). Utilisés en CI pour échouer si le code
n'est pas formaté correctement, sans modifier les fichiers.

### `bandit` — Sécurité

```ini
commands =
    bandit -r src
```

Scanne `src/` à la recherche de patterns dangereux : appels à `eval`, usages de modules dépréciés pour
la cryptographie, injections potentielles. L'analyse ne porte pas sur `tests/`.

---

## Environnements de correction automatique

### `black` et `isort` — Formatage automatique

```ini
commands =
    black --target-version py310 src tests
    isort src tests
```

Modifient les fichiers en place. À utiliser en local, jamais en CI.

### `format` — Formatage complet en une commande

```ini
commands =
    black --target-version py310 src tests
    isort src tests
```

Alias pratique pour appliquer black et isort en une seule invocation.

### `coverage` — Rapport de couverture

```ini
[testenv:coverage]
basepython = python3.12
commands =
    pytest --import-mode=importlib \
    --cov=<package_name> \
    --cov-append \
    --cov-report=term-missing \
    --cov-report=xml:{toxinidir}/coverage/coverage.xml \
    --cov-report=html:{toxinidir}/coverage/coverage_html tests
```

Génère trois formats de rapport :
- Terminal : lignes non couvertes visibles immédiatement
- XML (`coverage/coverage.xml`) : pour les outils CI (Codecov, SonarQube)
- HTML (`coverage/coverage_html/`) : pour la lecture locale interactive

`--cov-append` accumule les résultats si plusieurs runs sont enchaînés.

---

## Workflows complets

### `pre-push` — Porte qualité locale

```ini
[testenv:pre-push]
commands =
    black --target-version py310 src tests   # 1. formatage
    isort src tests
    bash -c "basedpyright src; ..."          # 2. types
    flake8 src tests                          # 3. lint
    bandit -r src                             # 4. sécurité
    bash .tox-config/scripts/test.sh         # 5. tests multi-versions
    bash .tox-config/scripts/coverage.sh     # 6. couverture
```

L'ordre est délibéré : le formatage passe en premier pour éviter les faux positifs de flake8.
En cas d'échec à n'importe quelle étape, l'exécution s'arrête.

```bash
tox -e pre-push
```

### `ci-quality` — Vérifications CI

Même logique que `pre-push` mais sans formatage automatique : utilise `--check` et `--check-only`.
Conçu pour échouer si le code soumis n'est pas conforme.

```bash
tox -e ci-quality
```

### `ci-tests` — Tests CI

Exécute uniquement pytest avec couverture. Utilisé par la matrice GitHub Actions pour tester chaque
version Python en parallèle.

```bash
tox -e ci-tests
```

---

## Configuration des outils

### flake8

```ini
[flake8]
max-line-length = 88
extend-ignore = E203, W503, E501
```

- `88` caractères : standard black
- `E203`, `W503`, `E501` : règles incompatibles avec le style black, ignorées

### isort

```ini
[isort]
profile = black
line_length = 88
known_first_party = <package_name>
multi_line_output = 3
include_trailing_comma = true
```

Le profil `black` garantit la compatibilité entre les deux outils.
`known_first_party` est à adapter avec le nom de votre paquet.

---

## Commandes de référence

```bash
tox -e pre-push          # workflow complet local
tox -e py312             # tests sur Python 3.12 uniquement
tox -e format            # formatage automatique
tox -e check             # vérification rapide sans modification
tox -e ci-quality        # vérification CI
tox -e ci-tests          # tests CI
tox -e coverage          # rapport de couverture seul
```

---

## Voir aussi

- [Configuration tox — English version](tox.en.md)
- [Script test.sh](../shell/tox-uv-test-script.fr.md)
- [Script coverage.sh](../shell/tox-uv-coverage-script.fr.md)
- [Wiki — Tests unitaires](https://gitlab.com/biface/biface/-/wikis/fr/controlled-delivery-software/test-management/testing)

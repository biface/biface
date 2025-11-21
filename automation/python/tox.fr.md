# Configuration Tox - Guide Complet

## Qu'est-ce que Tox ?

**Tox** est un outil d'automatisation des tests Python qui permet de :

- Créer des **environnements virtuels isolés** pour chaque configuration de test
- Tester votre code sur **plusieurs versions de Python** simultanément
- Exécuter des **suites de commandes standardisées** (tests, linting, formatage)
- Garantir la **reproductibilité** des tests entre développement local et CI/CD

### Pourquoi utiliser Tox ?

#### Pour les tests locaux

- **Isolation complète** : Chaque environnement de test est indépendant, évitant les conflits de dépendances
- **Cohérence** : Vous testez exactement dans les mêmes conditions que le CI/CD
- **Gain de temps** : Une seule commande (`tox`) pour lancer tous les tests et vérifications
- **Multi-versions** : Testez facilement votre code sur Python 3.9, 3.10, 3.11, et 3.12 en local

#### Pour le repository distant (CI/CD)

- **Standardisation** : Les mêmes commandes Tox fonctionnent en local et dans GitHub Actions
- **Maintenance simplifiée** : Modifier un test = modifier uniquement `tox.ini`, pas les workflows GitHub
- **Traçabilité** : Les développeurs peuvent reproduire exactement les tests du CI en local
- **Flexibilité** : Différents environnements Tox pour différents besoins (tests, qualité, sécurité)

---

## Configuration Générale

```ini
[tox]
minversion = 3.24.5
envlist = py39, py310, py311, py312
```

- **minversion** : Version minimale de Tox requise (3.24.5)
- **envlist** : Liste des environnements Python à tester par défaut (Python 3.9 à 3.12)

### Intégration GitHub Actions

```ini
[gh-actions]
python =
    3.9: py39
    3.10: py310
    3.11: py311
    3.12: py312
```

Cette section fait le **mapping** entre les versions Python de GitHub Actions et les environnements Tox. Lorsque GitHub Actions utilise Python 3.10, Tox active automatiquement l'environnement `py310`.

---

## Environnement de Test Principal

### [testenv] - Tests unitaires avec couverture

```ini
[testenv]
description = Run unit tests with coverage
```

Cet environnement par défaut s'applique à tous les environnements Python (py39, py310, py311, py312).

#### Configuration du PYTHONPATH

```ini
setenv =
    PYTHONPATH = {toxinidir}/src:{toxinidir}/tests
```

Définit le chemin Python pour que les imports fonctionnent correctement :

- `{toxinidir}` : Répertoire racine du projet
- Ajoute `src` et `tests` au PYTHONPATH

#### Dépendances

```ini
deps =
    pytest
    pytest-cov
```

Installe les outils nécessaires pour :

- **pytest** : Framework de tests
- **pytest-cov** : Plugin de mesure de couverture de code

#### Commandes d'exécution

```ini
commands =
    pytest --import-mode=importlib \
           --cov=ndict_tools \
           --cov-report=term-missing \
           --cov-report=xml:coverage/coverage.xml \
           --cov-report=html:coverage/coverage_html tests
```

Lance pytest avec :
- `--import-mode=importlib` : Mode d'import moderne et recommandé
- `--cov=ndict_tools` : Mesure la couverture du package `ndict_tools`
- `--cov-report=term-missing` : Affiche les lignes non couvertes dans le terminal
- `--cov-report=xml` : Génère un rapport XML (utilisé par Codecov)
- `--cov-report=html` : Génère un rapport HTML interactif

---

## Outils de Vérification Individuels

### [testenv:mypy] - Vérification des types

```ini
[testenv:mypy]
description = Type checking with mypy
```

Vérifie la cohérence des annotations de types Python pour détecter les erreurs de typage avant l'exécution.

### [testenv:flake8] - Linting

```ini
[testenv:flake8]
description = Lint the code with flake8
```

Analyse le code pour détecter :

- Erreurs de syntaxe
- Violations des conventions PEP 8
- Complexité excessive
- Code mort ou inutilisé

### [testenv:black-check] - Vérification du formatage

```ini
[testenv:black-check]
description = Check code formatting with black (no changes)
commands =
    black --check --diff src tests
```

Vérifie si le code respecte le formatage Black **sans modifier les fichiers** :

- `--check` : Mode vérification uniquement
- `--diff` : Affiche les différences si le formatage n'est pas correct

### [testenv:isort-check] - Vérification des imports

```ini
[testenv:isort-check]
description = Check import sorting with isort (no changes)
```

Vérifie que les imports sont triés selon les conventions **sans modifier les fichiers**.

### [testenv:bandit] - Analyse de sécurité

```ini
[testenv:bandit]
description = Run security analysis with bandit
commands =
    bandit -r src
```

Scanne le code à la recherche de vulnérabilités de sécurité courantes :

- Utilisation de fonctions dangereuses
- Problèmes de sécurité cryptographiques
- Injections potentielles
- `-r` : Analyse récursive

---

## Outils de Correction Automatique

### [testenv:black] - Formatage automatique

```ini
[testenv:black]
description = Auto-format code with black
commands =
    black src tests
```

Reformate automatiquement le code selon le style Black (sans `--check`).

### [testenv:isort] - Tri automatique des imports

```ini
[testenv:isort]
description = Auto-sort imports with isort
```

Trie automatiquement les imports selon les conventions.

### [testenv:format] - Formatage complet

```ini
[testenv:format]
description = Auto-format code (black + isort)
commands =
    black src tests
    isort src tests
```

Environnement combiné qui applique **à la fois** Black et isort pour un formatage complet.

---

## Workflows Complets

### [testenv:pre-push] - Workflow local avant push

```ini
[testenv:pre-push]
description = Complete local workflow before push (format + checks + tests)
deps =
    -r requirements.test.txt
    mypy
commands =
    # 1. Auto-formatting
    black src tests
    isort src tests
    
    # 2. Type checking
    mypy src
    
    # 3. Linting
    flake8 src tests
    
    # 4. Security
    bandit -r src
    
    # 5. Tests with coverage
    pytest --import-mode=importlib \
           --cov=ndict_tools \
           --cov-report=term-missing \
           --cov-report=xml:coverage/coverage.xml \
           --cov-report=html:coverage/coverage_html tests
```

Environnement **complet** à exécuter avant de pousser du code. Il combine :

1. **Auto-formatage** : Black + isort
2. **Vérification des types** : mypy
3. **Linting** : flake8
4. **Sécurité** : bandit
5. **Tests et couverture** : pytest avec les conditions de production des différents éléments vus plus haut.

**Usage** : `tox -e pre-push` avant chaque `git push` pour garantir la qualité.

### [testenv:ci-quality] - Vérifications CI (sans modification)

```ini
[testenv:ci-quality]
description = CI quality checks (no auto-fix, only verify)
deps =
    -r requirements.test.txt
    mypy
commands =
    # Vérifications sans modification
    mypy src
    flake8 src tests
    black --check --diff src tests
    isort --check-only --diff src tests
    bandit -r src
```

Version CI/CD qui **vérifie** sans modifier :

- Utilise `black --check` au lieu de `black`
- Utilise `isort --check-only` au lieu de `isort`

Parfait pour un pipeline CI qui doit échouer si le code n'est pas conforme.

### [testenv:ci-tests] - Tests CI

```ini
[testenv:ci-tests]
description = CI tests with coverage (used by gh-actions matrix)
deps =
    -r requirements.test.txt
commands =
    pytest --import-mode=importlib \
           --cov=ndict_tools \
           --cov-report=term-missing \
           --cov-report=xml:coverage/coverage.xml \
           --cov-report=html:coverage/coverage_html tests
```

Environnement simplifié utilisé par la matrice GitHub Actions. Exécute uniquement les tests avec couverture, sans les vérifications de qualité.

---

## Aliases Pratiques

### [testenv:check] - Vérification rapide

```ini
[testenv:check]
description = Quick check (no formatting, only verify)
```

Vérification rapide sans formatage automatique :

- mypy
- flake8
- black --check
- isort --check-only

**Usage** : `tox -e check` pour une vérification rapide sans modifier les fichiers.

### [testenv:local] - Alias de compatibilité

```ini
[testenv:local]
description = Alias pour pre-push (rétrocompatibilité)
```

Simple alias vers `pre-push` pour maintenir la compatibilité avec d'anciennes habitudes.

---

## Configuration des Outils

### Flake8

```ini
[flake8]
max-line-length = 88
extend-ignore = E203, W503, E501
```

- **max-line-length** : 88 caractères (standard Black)
- **extend-ignore** : Ignore les règles incompatibles avec Black
  - E203 : Espaces avant ':'
  - W503 : Saut de ligne avant opérateur binaire
  - E501 : Ligne trop longue (géré par Black)

### isort

```ini
[isort]
profile = black
line_length = 88
```

Configuration compatible avec Black :

- **profile = black** : Utilise le profil Black
- **line_length = 88** : Cohérent avec Black

### mypy

```ini
[mypy]
python_version = 3.9
warn_return_any = true
check_untyped_defs = true
ignore_missing_imports = true
```

Configuration de vérification des types :

- Cible Python 3.9 comme version minimale
- Active les avertissements sur les types incomplets
- Ignore les imports sans stubs de types

---

## Exemples d'Utilisation

### Tests locaux

```bash
# Tester sur toutes les versions Python
tox

# Tester sur Python 3.12 uniquement
tox -e py312

# Workflow complet avant push
tox -e pre-push

# Vérification rapide
tox -e check

# Formatage automatique
tox -e format
```

### Dans GitHub Actions

```yaml
# Le pipeline de tests utilise
run: tox -e ci-tests
```

```yaml
# Un pipeline de qualité pourrait utiliser
run: tox -e ci-quality
```

---

## Ressources complémentaires

- Fichier de configuration de tox [`tox.ini`](../../shared/tox_ini.txt)
- Fichier des packages requis [`requirements.test.txt`](../../shared/requirements.test.txt)

## Avantages de cette Configuration

1. **Environnements séparés** : Tests, qualité, sécurité sont isolés
2. **Flexibilité** : Choix de l'environnement selon le besoin
3. **Cohérence locale/distant** : Mêmes commandes partout
4. **Maintenance facilitée** : Un seul fichier à maintenir
5. **Documentation vivante** : Les descriptions expliquent chaque environnement

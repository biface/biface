# Pipeline de Tests Python (CI)

```yaml
name: Python CI - Tests
```

## Vue d'ensemble

Ce pipeline GitHub Actions exécute automatiquement les tests de votre projet Python pour garantir la qualité du code à chaque modification. Il vérifie la compatibilité avec plusieurs versions de Python.

## Déclenchement du pipeline

```yaml
on:
  push:
    branches:
      - "**"  # Toutes les branches (feature/*, hotfix/*, updates/*, staging/*)
  pull_request:
    branches:
      - "**"
```

Le pipeline se déclenche dans deux situations :

1. **À chaque push** sur n'importe quelle branche du dépôt (`"**"` signifie toutes les branches : `main`, `feature/*`, `hotfix/*`, `updates/*`, `staging-*`, etc.)
2. **À chaque pull request** créée ou mise à jour, quelle que soit la branche cible

## Architecture du pipeline

```yaml
jobs:
  test:
    name: Run tests on Python ${{ matrix.python-version }}
    runs-on: ubuntu-latest

    strategy:
      matrix:
        python-version: ["3.9", "3.10", "3.11", "3.12"]
```

### Stratégie de test matricielle

Le pipeline utilise une **matrice de tests** pour exécuter les tests sur plusieurs versions de Python simultanément :

- Python 3.9
- Python 3.10
- Python 3.11
- Python 3.12

Cela signifie que le job `test` sera exécuté **4 fois en parallèle**, une fois pour chaque version de Python, garantissant ainsi la compatibilité de votre code avec ces différentes versions.

L'utilisation d'une matrice vous permet :

- d'ajouter les nouvelles versions de python comme la version `"3.13"` ou la `"3.14"` (à date de novembre 2025)
- de retrancher des versions obsolètes comme `"3.2"`
- d'utiliser d'autres environnements d'exécution comme PyPy

Ici le choix est fait simplement de l'interpréteur standard de python.

```yaml
jobs:
  test:
    `[...]`
    strategy:
      matrix:
        python-version: ["3.9", "3.10", "3.11", "3.12"]
```

### Environnement d'exécution

```yaml
jobs:
  test:
    name: Run tests on Python ${{ matrix.python-version }}
    runs-on: ubuntu-latest
```

- **Le detail de l'exécution** du test est documenté `name:...` 
- **Système d'exploitation** : Ubuntu (dernière version disponible)
- **Runner** : Machine virtuelle fournie par GitHub Actions

## Étapes détaillées du pipeline

### Étape 1 : Récupération du code source
```yaml
- name: Checkout repository
  uses: actions/checkout@v4
```
Cette étape clone le dépôt Git dans l'environnement d'exécution. Elle utilise l'action officielle `checkout` en version 4, qui télécharge tout le code source nécessaire pour exécuter les tests.

### Étape 2 : Configuration de Python
```yaml
- name: Setup Python ${{ matrix.python-version }}
  uses: actions/setup-python@v4
  with:
    python-version: ${{ matrix.python-version }}
```
Cette étape installe la version spécifique de Python définie par la matrice. La variable `${{ matrix.python-version }}` prend successivement chaque valeur définie (3.9, 3.10, 3.11, 3.12) pour chaque exécution parallèle.

### Étape 3 : Installation des dépendances
```yaml
- name: Install dependencies
  run: |
    python -m pip install --upgrade pip
    pip install tox
```
Cette étape prépare l'environnement de test en :

1. Mettant à jour `pip` vers sa dernière version pour éviter les problèmes de compatibilité
2. Installant `tox`, un outil d'automatisation des tests qui gère les environnements virtuels et l'exécution des tests

### Étape 4 : Exécution des tests
```yaml
- name: Run tests with tox
  run: tox -e gh-ci
```
Cette dernière étape lance les tests en utilisant `tox` avec l'environnement `gh-ci` (GitHub CI). Tox va :

- Créer un environnement virtuel isolé
- Installer les dépendances de test définies dans votre configuration `tox.ini`
- Exécuter la suite de tests
- Rapporter les résultats (succès ou échec)

## Résultat

Si tous les tests passent sur les 4 versions de Python, le pipeline est marqué comme **réussi** ✅

![Détails du resultat de l'exécution réussie](../../pictures/python-test.png)

Si au moins un test échoue sur une version de Python, le pipeline est marqué comme **échoué** ❌, et vous recevrez une notification pour corriger le problème avant de fusionner le code.

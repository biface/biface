# Pipeline de Couverture de Code Python (CI)

```yaml
name: Python CI - Coverage
```

← [Pipelines](../../pipelines/pipelines.fr.md) | ← [Pipeline Tests](../../pipelines/tests.fr.md)

Le nom du pipeline importe pour le chaînage automatisé — c'est lui qui est référencé dans le `workflows:`
du déclencheur `workflow_run` qui suit dans la chaîne.

## Vue d'ensemble

Ce pipeline GitHub Actions mesure la couverture de code et télécharge automatiquement le rapport vers
Codecov. Il s'exécute **après** le pipeline [Tests](../../pipelines/tests.fr.md), et uniquement si celui-ci
a réussi — et uniquement sur les branches où la couverture a un sens pour la décision de release.

## Déclenchement du pipeline

```yaml
on:
  workflow_run:
    workflows: ["Python CI - Tests"]
    types:
      - completed
    branches:
      - "staging/**"   # release candidates
      - master         # production
```

### Conditions de déclenchement

1. **Pipeline parent** : se déclenche uniquement après l'exécution complète du pipeline "Python CI - Tests"
2. **Statut requis** : le pipeline de tests doit avoir réussi (`conclusion == 'success'`)
3. **Branches concernées** : uniquement `staging/**` (préparation de release) et la branche principale
   (production)

**Ce qui ne déclenche pas ce pipeline** : les branches `updates/X.Y.0` et `feature/*`. Sur ces branches, le
code est encore en développement — une mesure de couverture y serait partielle et sans valeur pour la
décision de release. Voir
[Tests d'intégration](https://gitlab.com/biface/biface/-/wikis/fr/controlled-delivery-software/test-management/integrate)
côté wiki pour la logique générale de restriction des étapes coûteuses aux bons moments du pipeline.

## Architecture du pipeline

```yaml
jobs:
  coverage:
    name: Upload coverage to Codecov
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
```

### Environnement d'exécution

- **Version Python** : 3.12 uniquement (contrairement au pipeline de tests, qui couvre cinq versions) — la
  couverture est mesurée sur une version de référence stable, pas sur toute la matrice. Voir
  [Configuration tox § coverage](../../tests/python/tox.fr.md#coverage--rapport-de-couverture) pour le
  raisonnement complet.

## Étapes détaillées du pipeline

### Étape 1 : Récupération du code source

```yaml
- name: Checkout repository
  uses: actions/checkout@v6
  with:
    ref: ${{ github.event.workflow_run.head_sha }}
```

Même nécessité que pour le pipeline Tests : ce pipeline est déclenché par `workflow_run`, il doit donc
explicitement préciser quel commit récupérer — `head_sha`, le commit exact qui a déclenché le pipeline
Tests, pas seulement le nom de la branche.

### Étape 2 : Configuration de Python

```yaml
- name: Setup Python
  uses: actions/setup-python@v5
  with:
    python-version: "3.12"
```

Une seule version, fixe — voir la justification ci-dessus.

### Étape 3 : Installation de uv et de tox

```yaml
- name: Install uv
  uses: astral-sh/setup-uv@v4

- name: Install tox
  run: uv pip install --system tox tox-uv
```

Identique aux pipelines Qualité et Tests.

### Étape 4 : Exécution des tests avec couverture

```yaml
- name: Run tests with coverage
  run: tox -e coverage
```

Réutilise l'environnement tox `coverage` — voir
[Configuration tox § coverage](../../tests/python/tox.fr.md#coverage--rapport-de-couverture). Important :
ce pipeline **réexécute** les tests, il ne réutilise pas les résultats du pipeline Tests précédent — chaque
job GitHub Actions tourne dans un environnement isolé et étanche, les résultats d'un job ne sont pas
directement accessibles à un autre sans étape explicite de transfert d'artefact.

### Étape 5 : Téléchargement vers Codecov

```yaml
- name: Upload coverage reports to Codecov
  uses: codecov/codecov-action@v5
  with:
    token: ${{ secrets.CODECOV_TOKEN }}
    fail_ci_if_error: true
```

Aucune condition supplémentaire sur cette étape : la restriction aux branches `staging/**`/`main` est déjà
posée au niveau du déclencheur du pipeline (`on.workflow_run.branches`) — il n'y a pas besoin de la répéter
ici.

`fail_ci_if_error: true` : un échec d'upload fait échouer tout le pipeline. Un seuil de couverture non
atteint ou une indisponibilité de Codecov est donc un signal bloquant, pas une simple alerte ignorée.

## Flux de travail complet

```text
1. Push sur une branche staging/** ou main
   ↓
2. Pipeline "Python CI - Quality" — doit réussir
   ↓
3. Pipeline "Python CI - Tests" — doit réussir
   ↓
4. Pipeline "Python CI - Coverage" se déclenche
   ↓
5. Téléchargement vers Codecov
```

## Résultat

- **Succès** ✅ : les tests passent et le rapport est téléchargé vers Codecov
- **Échec** ❌ : soit les tests échouent dans ce run, soit le téléchargement vers Codecov échoue

## Voir aussi

- [Pipeline Tests](../../pipelines/tests.fr.md) — déclencheur de celui-ci
- [Configuration tox § coverage](../../tests/python/tox.fr.md)
- [Coverage pipeline — English version](./coverage.en.md)

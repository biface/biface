# Pipeline de Qualité Python (CI)

```yaml
name: Python CI - Quality
```

← [Pipelines](./pipelines.md)

## Vue d'ensemble

Ce pipeline GitHub Actions est le **premier maillon** de la chaîne CI/CD. Il vérifie que le code respecte
les règles de base — typage, lint, formatage, sécurité — **avant** que tout autre pipeline ne se déclenche.
Le pipeline [Tests](./tests.fr.md) ne se lance que si celui-ci réussit : inutile de faire tourner une suite
de tests sur du code qui ne passerait même pas un contrôle de style.

## Déclenchement du pipeline

```yaml
on:
  push:
    branches:
      - '**'
  pull_request:
    branches:
      - '**'
```

Contrairement aux pipelines qui suivent dans la chaîne, celui-ci se déclenche directement sur **tout push et
toute pull request, sur toutes les branches** — c'est le seul point d'entrée de la chaîne CI. Tous les
pipelines suivants (Tests, Couverture) se déclenchent par cascade (`workflow_run`), pas par un événement Git
direct.

## Architecture du pipeline

```yaml
jobs:
  quality:
    name: Code Quality Checks
    runs-on: ubuntu-latest
```

Un seul job, qui enchaîne séquentiellement les cinq vérifications définies dans l'environnement tox
`ci-quality` — voir [Configuration tox](../tests/python/tox.fr.md#ci-quality--vérifications-ci) pour le
détail de chaque outil.

## Étapes détaillées du pipeline

### Étape 1 : Récupération du code source

```yaml
- name: Checkout code
  uses: actions/checkout@v6
```

Aucun paramètre `ref` particulier ici — ce pipeline se déclenche directement sur l'événement Git, il
récupère donc naturellement le bon commit. C'est seulement les pipelines suivants, déclenchés par
`workflow_run`, qui doivent préciser explicitement quel commit récupérer.

### Étape 2 : Configuration de Python

```yaml
- name: Set up Python
  uses: actions/setup-python@v5
  with:
    python-version: '3.12'
```

Une seule version de Python — la vérification de qualité (typage, lint, formatage) n'a pas besoin d'être
répétée sur toutes les versions supportées : le code source est le même quelle que soit la version qui
l'exécutera ensuite.

### Étape 3 : Installation de uv et de tox

```yaml
- name: Install uv
  uses: astral-sh/setup-uv@v4

- name: Install tox
  run: |
    uv pip install --system tox tox-uv
```

`uv` remplace `pip` pour l'installation des dépendances — plus rapide, en particulier sur des pipelines
exécutés fréquemment (à chaque push). `tox-uv` est le plugin qui permet à `tox` de déléguer ses installations
à `uv` plutôt qu'à `pip`.

### Étape 4 : Exécution des vérifications qualité

```yaml
- name: Run quality checks
  run: |
    tox -e ci-quality
```

`tox -e ci-quality` enchaîne, dans cet ordre, basedpyright (typage), flake8 (lint), black et isort en mode
vérification seule (formatage), bandit (sécurité) — voir
[Configuration tox](../tests/python/tox.fr.md#environnements-de-vérification) pour le détail de chaque outil.
Aucune correction automatique n'est appliquée ici : c'est un contrôle, pas une réparation. Si le code n'est
pas conforme, le pipeline échoue et signale précisément quel outil a échoué.

### Étape 5 : Conservation du rapport

```yaml
- name: Upload quality report
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: quality-report
    path: |
      .tox/ci-quality/log/
    retention-days: 30
```

`if: always()` : ce rapport est conservé **même en cas d'échec** — c'est précisément quand le pipeline a
échoué qu'on a besoin du détail des logs pour comprendre pourquoi. Conservé 30 jours, puis supprimé
automatiquement par GitHub.

## Résultat

Si les cinq vérifications passent, le pipeline est marqué **réussi** ✅ et déclenche automatiquement le
pipeline [Tests](./tests.fr.md) via `workflow_run`. Si l'une des cinq échoue, le pipeline est marqué
**échoué** ❌ — et aucun pipeline suivant ne se déclenche : tests, couverture et publication restent inactifs
jusqu'à ce que le code soit corrigé.

## Voir aussi

- [Pipeline Tests](./tests.fr.md) — déclenché par celui-ci
- [Configuration tox — environnement ci-quality](../tests/python/tox.fr.md)
- [Quality pipeline — English version](./quality.en.md)

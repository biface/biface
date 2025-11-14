# Pipeline de Couverture de Code Python (CI)

```yaml
name: Python CI - Coverage
```

Le nom du pipeline dans un objectif d'enchaînement automatisé de travaux importe, car il sera utilisé pour déclencher
ou non un autre flux de travail.

## Vue d'ensemble

Ce pipeline GitHub Actions mesure la couverture de code de votre projet Python et télécharge automatiquement
les rapports vers Codecov. Il s'exécute **après** le pipeline de tests et uniquement si celui-ci a réussi.

## Déclenchement du pipeline

Ce pipeline utilise un mécanisme de **déclenchement en cascade** (`workflow_run`), ce qui le rend dépendant d'un 
autre pipeline :

```yaml
on:
  workflow_run:
    workflows: ["Python CI - Tests"]
    types:
      - completed
    branches:
      - "updates/**"     # Toutes les branches de versions (updates/1.0.0, updates/1.1.0, etc.)
      - "staging/**"     # Toutes les branches de staging (staging/1.0.x, staging/1.1.x, etc.)
      - main
      - master

```

### Conditions de déclenchement

1. **Pipeline parent** : Se déclenche uniquement après l'exécution complète du workflow "Python CI - Tests"
2. **Statut requis** : Le pipeline de tests doit avoir réussi (`conclusion == 'success'`)
3. **Branches concernées** :
   - `updates/**` : Toutes les branches de versions (ex: `updates/1.0.0`, `updates/1.1.0`)
   - `staging/**` : Toutes les branches de staging (ex: `staging/1.0.x`, `staging/1.1.x`)
   - `main` : Branche principale
   - `master` : Branche principale alternative

**Important** : Ce pipeline ne se déclenche **jamais** directement. Il attend toujours que le pipeline de tests se
termine avec succès sur l'une des branches spécifiées.

## Architecture du pipeline

```yaml
jobs:
  coverage:
    name: Upload coverage to Codecov
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
```

### Condition d'exécution

```yaml
if: ${{ github.event.workflow_run.conclusion == 'success' }}
```

Le job `coverage` ne s'exécute que si le workflow parent (tests) s'est terminé avec succès. Si les tests échouent, 
ce pipeline est complètement ignoré. Cette approche permet d'optimiser les ressources consommées dans l'automatisation
des flux de travaux. Dans le cas présent, il ne **sert à rien** d'enclencher le traitement de la couverture du code sur 
le depôt GitHub en cas d'échec des tests. Dans cette hypothèse, il vaut mieux revenir sur les branches locales, pour
analyser, avec les erreurs publiées du précédent flux de travail, et les traiter. D'expérience, notamment avec
les mécanismes de complétion et d'aide à la production du code, les environnements de développement intégrés peuvent
ajouter des directives non souhaitées.

### Environnement d'exécution

- **Système d'exploitation** : Ubuntu (dernière version disponible)
- **Version Python** : 3.12 (version unique, contrairement au pipeline de tests)
- **Runner** : Machine virtuelle fournie par GitHub Actions

## Étapes détaillées du pipeline

### Étape 1 : Récupération du code source
```yaml
- name: Checkout repository
  uses: actions/checkout@v4
```
Clone le dépôt Git dans l'environnement d'exécution pour accéder au code source et aux fichiers de configuration.

### Étape 2 : Configuration de Python
```yaml
- name: Setup Python
  uses: actions/setup-python@v4
  with:
    python-version: "3.12"
```
Installe Python 3.12 spécifiquement. Contrairement au pipeline de tests qui utilise une matrice, ici une seule
version est nécessaire pour générer le rapport de couverture. Par contre, on peut se demander pourquoi refaire les
tests alors qu'ils ont déjà été réalisés précédemment : en fait, on ne peut pas reprendre les résultats du précédent
processus car ils sont étanches entre-eux.

### Étape 3 : Installation des dépendances
```yaml
- name: Install dependencies
  run: |
    python -m pip install --upgrade pip
    pip install tox
```
Prépare l'environnement en :
1. Mettant à jour `pip` vers la dernière version
2. Installant `tox` pour gérer l'exécution des tests avec mesure de couverture

### Étape 4 : Exécution des tests avec couverture
```yaml
- name: Run tests with coverage
  run: tox -e gh-ci
```
Lance les tests en utilisant le même environnement `gh-ci` que le pipeline de tests, mais cette fois-ci en générant également des données de couverture de code (pourcentage de code testé).

### Étape 5 : Téléchargement vers Codecov
```yaml
- name: Upload coverage reports to Codecov
  if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/master'
  uses: codecov/codecov-action@v5
  with:
    token: ${{ secrets.CODECOV_TOKEN }}
    fail_ci_if_error: true
```

Cette étape finale télécharge le rapport de couverture vers Codecov, mais **uniquement** pour les branches principales (`main` ou `master`).

**Paramètres importants** :
- `token` : Utilise un jeton secret stocké dans les secrets GitHub pour s'authentifier auprès de Codecov
- `fail_ci_if_error: true` : Si le téléchargement échoue, le pipeline entier échoue également, garantissant qu'on détecte les problèmes de reporting

**Comportement selon la branche** :
- Sur `main` ou `master` : Le rapport est téléchargé vers Codecov
- Sur `updates/**` ou `staging/**` : Les tests sont exécutés avec couverture, mais le rapport n'est **pas** téléchargé (pour éviter de polluer Codecov avec des branches temporaires)

## Flux de travail complet

```
1. Push de code sur une branche concernée
   ↓
2. Pipeline "Python CI - Tests" se déclenche et s'exécute
   ↓
3. Si les tests réussissent → Pipeline "Coverage" se déclenche
   ↓
4. Exécution des tests avec mesure de couverture
   ↓
5. Si branche = main/master → Téléchargement vers Codecov
```

## Résultat

- **Succès** ✅ : Les tests passent avec succès et le rapport de couverture est généré (et téléchargé si sur `main`/`master`)
- **Échec** ❌ : Soit les tests échouent, soit le téléchargement vers Codecov échoue (uniquement sur `main`/`master`)
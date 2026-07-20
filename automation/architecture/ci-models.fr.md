# Deux architectures de pipeline

← [Automatisation des tests](../automation.fr.md) | _English version → [ci-models.en.md](ci-models.en.md)_

## Vue d'ensemble

Tous les pipelines documentés dans ce dépôt suivent l'une de ces deux architectures. Le choix entre les
deux n'a **rien à voir avec le langage** du projet — Python et Rust cohabitent dans chacune des deux
familles. Ce qui distingue réellement ces architectures, c'est **comment les étapes s'enchaînent** :

- **chaînage événementiel multi-fichiers** (`workflow_run`) — un fichier de workflow par étape, chaque
  fichier se déclenchant sur l'achèvement du précédent ;
- **chaînage de dépendances mono-fichier** (`needs:`) — un seul fichier de workflow, dont les jobs
  déclarent explicitement leurs dépendances internes.

Un projet donné adopte l'une ou l'autre — pas les deux — mais le même projet peut ensuite héberger un
troisième workflow totalement indépendant (couverture, publication de documentation, miroir de dépôt…) qui
se déclenche seul, hors de toute chaîne. Ce cas est traité à part dans chaque implémentation.

## Modèle 1 — chaînage événementiel (`workflow_run`)

```yaml
# fichier B : se déclenche à la fin du fichier A
on:
  workflow_run:
    workflows: ["<nom du workflow A>"]
    types:
      - completed
    branches:
      - "**"

jobs:
  <job>:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
      - uses: actions/checkout@v6
        with:
          ref: ${{ github.event.workflow_run.head_sha }}
```

Chaque étape de la chaîne vit dans son **propre fichier** `.github/workflows/*.yaml`, avec son propre nom
de workflow. GitHub Actions les relie via le déclencheur `workflow_run`, qui écoute la fin d'un autre
workflow nommé explicitement.

**Points d'attention propres à ce modèle :**

- le fichier déclenché par `workflow_run` ne reçoit pas automatiquement le bon commit — il faut préciser
  `ref: ${{ github.event.workflow_run.head_sha }}` au checkout, sans quoi GitHub récupère la branche par
  défaut ;
- la condition `if: github.event.workflow_run.conclusion == 'success'` doit être répétée dans **chaque**
  fichier suivant — l'oubli fait tourner le job même si l'étape précédente a échoué ;
- chaque fichier est visible séparément dans l'onglet Actions, ce qui donne une lecture plus fine de
  l'historique (un badge par étape), au prix d'un couplage par nom de chaîne de caractères entre fichiers
  (renommer un workflow casse silencieusement la chaîne).

→ implémentation documentée : [pipelines/workflow-run/](../pipelines/workflow-run/pipelines.fr.md)

## Modèle 2 — chaînage de dépendances (`needs:`)

```yaml
# un seul fichier, tous les jobs
jobs:
  <job_a>:
    steps: [...]

  <job_b>:
    needs: <job_a>
    steps: [...]

  <job_c>:
    needs: <job_b>
    if: startsWith(github.ref, 'refs/tags/')
    steps: [...]
```

Toutes les étapes vivent dans un **seul fichier**. GitHub Actions calcule le graphe d'exécution à partir
des `needs:` déclarés par chaque job — un job attend que tous ceux listés dans son `needs:` aient réussi
avant de démarrer.

**Points d'attention propres à ce modèle :**

- le commit est correct par construction (un seul événement Git déclenche tout le fichier) — pas de
  `head_sha` à gérer ;
- la restriction d'un job à certaines conditions (par exemple : ne construire/publier que sur tag) se fait
  avec `if:`, indépendamment du `needs:` — les deux se combinent ;
- un seul badge de statut pour l'ensemble de la chaîne dans l'onglet Actions ; le détail par étape se lit
  dans les jobs du run, pas dans une liste de workflows séparés ;
- `needs:` accepte une liste (`needs: [job_a, job_b]`) quand un job dépend de plusieurs prédécesseurs — ce
  que `workflow_run` ne permet pas d'exprimer nativement.

→ implémentation documentée : [pipelines/needs/](../pipelines/needs/pipeline.fr.md), avec deux exemples
réels (une chaîne longue et une chaîne courte) pour montrer que la longueur de la chaîne — pas le
langage — est ce qui varie d'un projet à l'autre.

## Lequel choisir ?

| | `workflow_run` | `needs:` |
|---|---|---|
| Nombre de fichiers | un par étape | un seul |
| Lecture du run | un badge par étape | un badge global, détail par job |
| Risque de couplage | par nom de workflow (chaîne de caractères) | aucun, tout est dans le même fichier |
| Ajout d'une étape | nouveau fichier + déclencheur `workflow_run` | nouveau job + `needs:` |
| Dépendances multiples | non exprimable nativement | `needs: [a, b]` |

Le modèle mono-fichier (`needs:`) est celui recommandé pour les nouveaux projets — voir DD-36 dans les
design decisions du projet concerné. Le modèle multi-fichiers reste documenté et maintenu là où il est déjà
en place, sans migration forcée.

---

## Licence

[CC BY-NC 4.0](../../LICENSE.txt) — documentation et fichiers de configuration.

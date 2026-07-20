# Documentation des pipelines CI/CD

← [Automatisation des tests](../automation.fr.md)

## Introduction

Cette documentation explique en détail les workflows GitHub Actions (ou simplement « pipelines »)
utilisés dans mes projets. Ces pipelines automatisent les aspects critiques du cycle de développement :
qualité du code, tests, couverture, et build/publication.

**Note :** _English version → [pipelines.en.md](pipelines.en.md)_

## Objectif

Les pipelines décrits dans cette documentation répondent à plusieurs objectifs :

- **Automatisation** : réduire l'intervention manuelle en automatisant les tâches répétitives, [comme les
  tests](../automation.fr.md), et le build/publication
- **Assurance qualité** : garantir la qualité et la compatibilité du code avant toute fusion
- **Intégration continue** : valider chaque changement automatiquement pour détecter les problèmes tôt
- **Cohérence** : maintenir des processus standardisés sur toutes les branches
- **Fiabilité** : fournir un retour rapide aux développeurs sur l'état de leurs changements

## Deux architectures

Les pipelines de ce dépôt suivent l'une de deux architectures — voir
[Deux architectures de pipeline](../architecture/ci-models.fr.md) pour le détail conceptuel et le
comparatif complet. En résumé :

1. **[workflow-run/](workflow-run/pipelines.fr.md)** — chaînage événementiel multi-fichiers
   (`workflow_run`). Chaque étape (Qualité, Tests, Couverture) vit dans son propre fichier de workflow,
   déclenché par l'achèvement du précédent.
2. **[needs/](needs/pipeline.fr.md)** — chaînage de dépendances mono-fichier (`needs:`). Toutes les étapes
   vivent dans un seul fichier, avec deux implémentations documentées côte à côte pour montrer que c'est
   la longueur de la chaîne — pas le langage — qui varie d'un projet à l'autre.

## Comment utiliser cette documentation

Chaque pipeline est documenté avec :

- Une vue d'ensemble de son objectif
- Les conditions de déclenchement (quand il s'exécute)
- Une explication détaillée étape par étape de chaque action
- Les résultats attendus

Parcourez les deux sous-dossiers ci-dessus pour explorer chaque architecture en détail. La configuration
`tox` partagée par les pipelines Python est documentée séparément dans
[Configuration tox](../tests/python/tox.fr.md).

# Documentation des pipelines CI/CD

## Introduction

Cette documentation explique en détail les workflows GitHub Actions (ou simplement « pipelines »)
utilisés dans mes projets. Ces pipelines automatisent les aspects critiques du cycle de développement :
qualité du code, tests, couverture, et — à terme — déploiement.

**Note :** _English version → [pipelines.en.md](pipelines.en.md)_

## Objectif

Les pipelines décrits dans cette documentation répondent à plusieurs objectifs :

- **Automatisation** : réduire l'intervention manuelle en automatisant les tâches répétitives, [comme les
  tests](../automation.md), et à terme le déploiement
- **Assurance qualité** : garantir la qualité et la compatibilité du code avant toute fusion
- **Intégration continue** : valider chaque changement automatiquement pour détecter les problèmes tôt
- **Cohérence** : maintenir des processus standardisés sur toutes les branches
- **Fiabilité** : fournir un retour rapide aux développeurs sur l'état de leurs changements

## Vue d'ensemble des pipelines

Les pipelines suivants sont documentés dans ce dépôt, dans l'ordre où ils s'enchaînent :

1. **Python CI - Quality** → [quality.fr.md](quality.fr.md) — vérifications de typage, lint, formatage et
   sécurité, premier maillon de la chaîne ; tout pipeline suivant en dépend
2. **Python CI - Tests** → [tests.fr.md](tests.fr.md) — exécution de la suite de tests sur plusieurs
   versions Python, déclenchée uniquement si Quality a réussi
3. **Python CI - Coverage** → [../coverage/python/coverage.fr.md](../coverage/python/coverage.fr.md) —
   mesure de la couverture de code et publication vers Codecov, déclenchée uniquement sur `staging/**` et
   la branche principale

_(Les pipelines de build et de publication seront documentés dans une prochaine itération.)_

## Comment utiliser cette documentation

Chaque pipeline est documenté avec :

- Une vue d'ensemble de son objectif
- Les conditions de déclenchement (quand il s'exécute)
- Une explication détaillée étape par étape de chaque action
- Les résultats attendus

Parcourez les pages ci-dessus pour explorer chaque pipeline en détail. La configuration `tox` partagée par
ces trois pipelines est documentée séparément dans
[Configuration tox](../tests/python/tox.fr.md).

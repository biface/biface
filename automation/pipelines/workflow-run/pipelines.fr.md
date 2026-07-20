# Pipelines — chaînage événementiel multi-fichiers (`workflow_run`)

← [Pipelines](../pipelines.fr.md) | _English version → [pipelines.en.md](pipelines.en.md)_

Implémentation de référence : [ndt](https://github.com/biface/ndt) (Python). Trois fichiers de workflow
distincts, chacun déclenché par l'achèvement du précédent :

1. **[Qualité](quality.fr.md)** — premier maillon, se déclenche directement sur push/pull request
2. **[Tests](tests.fr.md)** — se déclenche par `workflow_run` sur l'achèvement de Qualité
3. **[Couverture](../../coverage/python/coverage.fr.md)** — se déclenche par `workflow_run` sur
   l'achèvement de Tests, uniquement sur `staging/**` et la branche principale

Le principe général du modèle (déclencheur `workflow_run`, gestion du `head_sha`, répétition de la
condition `conclusion == 'success'`) est expliqué dans
[Deux architectures de pipeline](../../architecture/ci-models.fr.md#modèle-1--chaînage-événementiel-workflow_run).
Les trois pages ci-dessus documentent chacune l'implémentation concrète d'une étape.

Une quatrième page, **[Tests — modèle renforcé](test-uv.fr.md)**, documente à titre d'exemple une approche
alternative de l'étape Tests (gestion de plusieurs environnements via `.tox-config`) — abandonnée, conservée
pour référence, non reprise dans `shared/github-ci/`.

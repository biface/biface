# Automatisation des tests

← [README](../README.md) | _English version → [automation.md](automation.md)_

## Vue d'ensemble

L'automatisation des tests consiste à utiliser des outils logiciels pour exécuter les tests automatiquement,
fournissant un retour rapide sur la qualité du code sans intervention manuelle. Cette automatisation est
essentielle dans les workflows de développement modernes, en particulier dans les pipelines CI/CD où chaque
changement de code doit être validé rapidement et de façon fiable.

## Développement piloté par les tests (TDD)

Le **Test-Driven Development** est une méthodologie qui place les tests automatisés au cœur même du
processus de développement. Plutôt que d'écrire les tests après le code, le TDD inverse cette relation.

### Le cycle TDD : Red-Green-Refactor

```mermaid
flowchart LR
    P(["Écrire un test<br>(Red)"]) --> n1(["Écrire le code<br>(Green)"]) --> n2(["Recommencer<br>(Refactor)"])
```

1. **Red** : écrire un test qui échoue, définissant le comportement attendu
2. **Green** : écrire le code minimal nécessaire pour faire passer le test
3. **Refactor** : améliorer le code en gardant les tests au vert

### Pourquoi le TDD profite à l'automatisation

- **Couverture naturelle** : chaque ligne de code de production répond à un test — la couverture atteint
  naturellement des niveaux élevés, sans cas oublié
- **Meilleure conception** : écrire le test d'abord force à penser interfaces et API avant l'implémentation
- **Documentation vivante** : les tests décrivent ce que le code doit faire, toujours à jour
- **Confiance dans le refactoring** : une suite de tests complète détecte immédiatement les régressions
- **Débogage plus rapide** : les échecs sont détectés au moment même où le code est écrit

## Tests unitaires et tests d'intégration

### Tests unitaires — le socle

Testent un composant isolé (fonction, méthode, classe), s'exécutent en millisecondes, ne nécessitent aucune
dépendance externe. Rapides, précis dans la localisation d'un échec, parallélisables.

### Tests d'intégration — les interactions entre composants

Vérifient que plusieurs composants fonctionnent correctement ensemble : flux de données entre modules,
contrats d'API respectés. Plus lents que les tests unitaires, plus rapides que les tests de bout en bout.

```text
Changement de code
    ↓
Commit & Push
    ↓
Pipeline CI déclenché
    ↓
1. Qualité (secondes)        ────→ retour immédiat
    ↓
2. Tests unitaires (minutes) ────→ validation des composants
    ↓
3. Rapport de couverture      ────→ indicateur de qualité
    ↓
Notification succès/échec
```

---

## Structure de ce dépôt

```text
biface/biface
├── shared/                        ← fichiers de configuration réutilisables
│   ├── tox.ini                    ← configuration tox de référence
│   ├── tox-config/                ← configuration d'utilisation de tox
│   │   ├── requirements/
│   │   ├── versions.txt
│   │   ├── coverage-version.txt
│   │   └── scripts/
│   │       ├── test.sh
│   │       └── coverage.sh
│   └── issue-templates/
│       └── github/                ← gabarits d'issues GitHub
└── automation/
    ├── automation.fr.md           ← cette page
    ├── pipelines/
    │   ├── pipelines.fr.md        ← vue d'ensemble des pipelines GitHub Actions
    │   ├── quality.fr.md          ← pipeline Python CI - Quality
    │   └── tests.fr.md            ← pipeline Python CI - Tests
    ├── coverage/python/
    │   └── coverage.fr.md         ← pipeline Python CI - Coverage
    └── tests/
        ├── python/
        │   └── tox.fr.md          ← configuration tox expliquée
        └── shell/
            ├── tox-uv-test-script.fr.md      ← script test.sh expliqué
            └── tox-uv-coverage-script.fr.md  ← script coverage.sh expliqué
```

## Comment utiliser ce dépôt

Les fichiers de `shared/` sont conçus pour être copiés dans n'importe quel projet Python et adaptés avec un
minimum de modifications — voir [Configuration tox](tests/python/tox.fr.md) pour la procédure complète
(copie de `tox.ini`, des `requirements/`, des scripts, adaptation du nom de paquet).

Les pipelines GitHub Actions qui consomment cette configuration sont documentés dans
[pipelines/pipelines.fr.md](pipelines/pipelines.fr.md) — c'est là que se trouve le détail de comment chaque
workflow `.github/workflows/*.yaml` invoque `tox` et dans quel ordre.

---

## Validé sur

| Projet | Registre |
| --- | --- |
| [ndt](https://github.com/biface/ndt) | [PyPI](https://pypi.org/project/ndict-tools/) |
| [sds](https://github.com/skyfrigate/sds) | — |
| [i18n](https://github.com/biface/i18n) | — |

---

## Documentation wiki

Le wiki explique le **pourquoi** et le **quoi** ; ce dépôt montre le **comment sur github**.

- [Tests — vue d'ensemble](https://gitlab.com/biface/biface/-/wikis/fr/controlled-delivery-software/test-management)
- [Tests unitaires](https://gitlab.com/biface/biface/-/wikis/fr/controlled-delivery-software/test-management/testing)
- [Couverture de code](https://gitlab.com/biface/biface/-/wikis/fr/controlled-delivery-software/test-management/coverage)

---

## Licence

[CC BY-NC 4.0](../LICENSE.txt) — documentation et fichiers de configuration.

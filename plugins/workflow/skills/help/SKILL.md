---
name: help
description: Sommaire du workflow de développement GMU — présente les deux tracks (standard et fast), tous les skills * avec leur ordre d'utilisation, les artifacts produits et les commandes QA. Déclenche dès que l'utilisateur demande "comment je commence", "quel skill utiliser", "rappelle-moi le workflow", "c'est quoi quoi", ou est perdu dans le pipeline, même sans citer le skill.
user_invocable: true
---

# /help — Guide du workflow de développement GMU

## Schéma global du pipeline

```
                       TRACK STANDARD (features structurantes)
   ┌──────────┐   ┌──────────┐   ┌────────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌────────┐
   │feature│──▶│design │──▶│implement│──▶│review │──▶│commit │──▶│report │──▶│sync │
   └────┬─────┘   └────┬─────┘   └─────┬──────┘   └────┬─────┘   └────┬─────┘   └────┬─────┘   └───┬────┘
        │              │               │               │              │              │             │
     feature.md    design.md      code+migrations  review.md       commit         report.md     doc sync
                                  +nouveaux tests                  + push                       + changelog

                           UTILITAIRES (hors pipeline, à la demande)
   ┌────────────────┐    ┌──────────────────┐    ┌──────────┐
   │ doc-sylius  │    │ test-scenario │    │ help  │  ← tu y es
   └────────────────┘    └──────────────────┘    └──────────┘
   docs sylius-native     Playwright MCP live    ce sommaire
```

Règle d'or : ne jamais passer à l'étape suivante sans validation explicite du user ("ok", "go", "validé", "c", etc.).

## Track standard — Features et changements structurants

Pour tout changement qui implique une migration, un nouveau service, un impact multi-channel, ou qui touche > 3 fichiers.

| #  | Skill              | Rôle                                                         | Produit                              |
|----|--------------------|--------------------------------------------------------------|--------------------------------------|
| 1  | `/feature`      | Cadrer et challenger une fonctionnalité                      | `docs/features/NNN-slug/feature.md`  |
| 2  | `/design`       | Concevoir la solution technique à partir de la spec          | `docs/features/NNN-slug/design.md`   |
| 3  | `/implement`    | Implémenter sous-tâche par sous-tâche avec QA continue       | Code + migrations + tests            |
| 4  | `/review`       | Code review du diff (sécu, perf, qualité, conformité design) | `docs/features/NNN-slug/review.md`   |
| 5  | `/commit`       | Commit Conventional Commits en français + push               | Commit sur main                      |
| 6  | `/report`       | Documenter ce qui a été fait vs ce qui était prévu           | `docs/features/NNN-slug/report.md`   |
| 7  | `/sync`         | Réaligner spec et design avec la réalité du code             | Mise à jour `feature.md` + `design.md` |

## Track fast — Bugfixes et petits changements

**Conditions (toutes requises)** :

- Moins de 3 fichiers modifiés
- Pas de migration Doctrine
- Pas de nouveau service ou entité
- Pas d'impact multi-channel / multi-thème
- Pas de changement d'API publique

**Processus** :

```
coder → QA (ECS + PHPStan + build) → tests ciblés → /review (optionnel) → /commit
```

En cas de doute → partir sur le track standard. Il est toujours possible de basculer du standard vers le fast si l'analyse révèle que c'est trivial.

## Utilitaires (hors pipeline)

| Skill                | Rôle                                                                              |
|----------------------|-----------------------------------------------------------------------------------|
| `/doc-sylius`     | Analyser et documenter une feature Sylius existante → `docs/sylius-native/NNN-slug/` |
| `/test-scenario`  | Tester un scénario utilisateur via Playwright MCP (navigateur piloté en live)     |
| `/help`           | Ce sommaire — pour se rappeler le workflow et les skills disponibles              |

## Commandes de référence

```bash
# QA obligatoire (à lancer après chaque sous-tâche en track standard, et une fois en fin de track fast)
symfony php vendor/bin/ecs check --fix
symfony php vendor/bin/phpstan analyse
npm run build

# Tests
symfony php vendor/bin/phpunit
npm run test:e2e
npm run test:e2e:{feature}    # par famille (brands, bundles, etc.)

# Migrations
symfony console make:migration
symfony console doctrine:migrations:migrate --dry-run
symfony console doctrine:migrations:migrate
symfony console doctrine:schema:validate
```

## Règles permanentes (rappel)

- Ne jamais modifier le vendor — surcharger via les mécanismes officiels Sylius
- Toute modification de schéma = migration Doctrine **générée** et relue
- Ne jamais modifier une migration déjà commitée (en créer une nouvelle)
- Snake_case en BDD : `#[ORM\Column(name: 'mon_champ')]` pour tout champ camelCase en PHP
- DQL/SQL uniquement dans les repositories (`src/Repository/`)
- Validation custom Sylius : `groups: ['Default', 'sylius']` obligatoire
- FormTypeExtension + Twig Hooks symétriques (sinon 422 silencieux)
- Composants Twig en sous-dossier : `Media:MonComposant` (namespace préfixé par `:`)
- Multi-thème = front shop uniquement (pas l'admin)
- Checkpoints humains entre chaque étape du pipeline
- Un commit = une intention (Conventional Commits v1.0.0 en français)

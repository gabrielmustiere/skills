---
name: help
description: Sommaire du workflow de développement — présente les deux tracks (standard et fast), tous les skills avec leur ordre d'utilisation et les artifacts produits. Déclenche sur "je suis perdu", "quoi faire maintenant ?", "quel skill utiliser ?", "rappelle-moi le workflow", "par où je commence ?", "c'est quoi <skill>" — même sans citer le skill.
user_invocable: true
---

# /help — Guide du workflow de développement

## Schéma global du pipeline

```
                       TRACK STANDARD (features structurantes)
   ┌─────────┐   ┌────────┐   ┌───────────┐   ┌────────┐   ┌────────┐   ┌────────┐   ┌──────┐
   │feature │──▶│design │──▶│implement│──▶│review │──▶│commit │──▶│report │──▶│sync │
   └────┬────┘   └───┬────┘   └────┬──────┘   └───┬────┘   └───┬────┘   └───┬────┘   └──┬───┘
        │            │             │              │            │            │           │
    feature.md   design.md    code+migrations   review.md    commit       report.md   doc sync
                              +nouveaux tests                + push                   + changelog

                           UTILITAIRES (hors pipeline, à la demande)
   ┌─────────────────┐    ┌──────┐
   │ test-scenario │    │ help │  ← tu y es
   └─────────────────┘    └──────┘
   Playwright MCP live    ce sommaire
```

Règle d'or : ne jamais passer à l'étape suivante sans validation explicite du user ("ok", "go", "validé", "c", etc.).

## Track standard — Features et changements structurants

Pour tout changement qui implique une migration, un nouveau service, un impact transverse, ou qui touche plus de quelques fichiers.

| #  | Skill         | Rôle                                                         | Produit                                |
|----|---------------|--------------------------------------------------------------|----------------------------------------|
| 1  | `/feature`    | Cadrer et challenger une fonctionnalité                      | `docs/features/NNN-slug/feature.md`    |
| 2  | `/design`     | Concevoir la solution technique à partir de la spec          | `docs/features/NNN-slug/design.md`     |
| 3  | `/implement`  | Implémenter sous-tâche par sous-tâche avec QA continue       | Code + migrations + tests              |
| 4  | `/review`     | Code review du diff (sécu, perf, qualité, conformité design) | `docs/features/NNN-slug/review.md`     |
| 5  | `/commit`     | Commit Conventional Commits en français + push               | Commit                                 |
| 6  | `/report`     | Documenter ce qui a été fait vs ce qui était prévu           | `docs/features/NNN-slug/report.md`     |
| 7  | `/sync`       | Réaligner spec et design avec la réalité du code             | Mise à jour `feature.md` + `design.md` |

## Track fast — Bugfixes et petits changements

**Conditions (toutes requises)** :

- Moins de 3 fichiers modifiés
- Pas de changement de schéma (migration) ni de nouveau service/entité
- Pas d'impact transverse (multi-channel, multi-thème, API publique…)

**Processus** :

```
coder → QA du stack → tests ciblés → /review (optionnel) → /commit
```

En cas de doute → partir sur le track standard. Il est toujours possible de basculer du standard vers le fast si l'analyse révèle que c'est trivial.

## Utilitaires (hors pipeline)

| Skill            | Rôle                                                                              |
|------------------|-----------------------------------------------------------------------------------|
| `/test-scenario` | Tester un scénario utilisateur via Playwright MCP (navigateur piloté en live)     |
| `/help`          | Ce sommaire — pour se rappeler le workflow et les skills disponibles              |

Des plugins complémentaires (ex: `sylius`, `symfony`) peuvent exposer des skills plus tactiques (procédures spécifiques au framework : créer une Resource, diagnostiquer un Twig Hook, etc.). Ils se combinent naturellement avec ce workflow via l'auto-découverte de Claude Code.

## Règles framework

Le workflow détecte automatiquement le stack du projet (Symfony, Sylius) via `composer.json` / `package.json` et charge les règles correspondantes. Voir :

- `plugins/workflow/references/stacks/_detection.md` — procédure de détection
- `plugins/workflow/references/stacks/symfony.md` — règles Symfony (Doctrine, services, forms, Twig, QA, sécu, perf)
- `plugins/workflow/references/stacks/sylius.md` — delta e-commerce Sylius (Resources, channels, thèmes, Twig Hooks…)

Les conventions propres à ton projet (commandes QA exactes, credentials de test, noms de thèmes utilisés, branches…) vivent dans le `CLAUDE.md` à la racine du projet — les skills le lisent en complément des références stack.

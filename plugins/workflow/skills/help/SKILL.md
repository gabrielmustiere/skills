---
name: help
description: Affiche le sommaire du workflow et oriente vers le bon skill à appeler — détaille les phases amont (vision, product-backlog), les trois tracks d'exécution (feature, refacto, tech), l'enchaînement des skills dans chaque track et les artifacts produits à chaque étape. À utiliser quand tu ne sais pas par où commencer ou quel skill correspond à ton besoin.
user_invocable: true
disable-model-invocation: true
model: haiku
allowed-tools:
  - Read
  - Glob
---

# /help — Guide du workflow de développement

## Schéma global du pipeline

```
                       PHASE 0 — VISION PROJET (une fois, ou pivot)
                       ┌────────┐
                       │vision │──▶ docs/vision.md (problème, audience, valeur,
                       └────────┘    North Star, principes, anti-objectifs)
                                     Lu par product-backlog puis feature-pitch.

                       PHASE 0.5 — PÉRIMÈTRE FONCTIONNEL & BACKLOG
                       ┌──────────────┐
                       │product-backlog│──▶ docs/product-backlog.md (domaines, capacités,
                       └──────────────┘    parcours, règles transverses, backlog priorisé MVP/V2/V3)
                                           Lu par feature-pitch pour situer chaque feature.

                        TRACK FEATURE (valeur utilisateur, structurante)
 ┌──────────────┐   ┌───────────────┐   ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐   ┌──────┐
 │feature-pitch │──▶│feature-design│──▶│feature│──▶│review │──▶│commit │──▶│report │──▶│sync │
 └──────┬───────┘   └───────┬───────┘   └───┬────┘   └───┬────┘   └───┬────┘   └───┬────┘   └──┬───┘
        │                    │              │            │            │            │           │
        feature.md        design.md   code+migrations review.md    commit       report.md   doc sync
                                      +nouveaux tests              +push                   +changelog

                    TRACK REFACTO (comportement figé, code restructuré)
 ┌──────────────┐   ┌─────────┐   ┌────────┐   ┌────────┐   ┌────────┐   ┌──────┐
 │refactor-plan │──▶│refactor│──▶│review │──▶│commit │──▶│report │──▶│sync │
 └──────┬───────┘   └────┬────┘   └───┬────┘   └───┬────┘   └───┬────┘   └──┬───┘
        │                │             │            │            │           │
        plan.md     verrou tests    review.md    commit        report.md   doc sync
                    + étapes                     +push
                    incrémentales

               TRACK TECH (perf, résilience, observabilité, sécu — non user-facing)
 ┌──────────┐   ┌─────┐   ┌────────┐   ┌────────┐   ┌────────┐   ┌──────┐
 │tech-plan│──▶│tech │──▶│review │──▶│commit │──▶│report │──▶│sync │
 └────┬─────┘   └──┬──┘   └───┬────┘   └───┬────┘   └───┬────┘   └──┬───┘
      │            │           │            │            │           │
      plan.md   baseline    review.md   commit       report.md   doc sync
                + kill switch           +push
                + étapes mesurées

                       UTILITAIRES (hors pipeline, à la demande)
 ┌─────────────────┐    ┌──────┐
 │ test-scenario │    │ help │  ← tu y es
 └─────────────────┘    └──────┘
 Playwright MCP live    ce sommaire
```

Règle d'or : ne jamais passer à l'étape suivante sans validation explicite du user ("ok", "go", "validé", "c", etc.).

## Convention dossiers `docs/story/`

Tous les artifacts vivent dans `docs/story/` à plat, **numérotés globalement** puis taggés par type. Format : `NNN-<f|r|t>-<slug>`. Le numéro vient en premier pour que le tri lexicographique de `ls` corresponde à l'ordre chronologique.

| Tag    | Type              | Doc d'intention              | Exemple                             |
|--------|-------------------|------------------------------|-------------------------------------|
| `f`    | Feature           | `feature.md` + `design.md`  | `docs/story/042-f-checkout-express/` |
| `r`    | Refacto           | `plan.md`                    | `docs/story/043-r-extract-pricing/`  |
| `t`    | Évolution tech    | `plan.md`                    | `docs/story/044-t-redis-cache/`      |

Les numéros s'incrémentent globalement (042-f → 043-r → 044-t → 045-f…), ce qui permet de lire la timeline d'évolution du projet en listant simplement `docs/story/`.

## Phase 0 — Vision projet

Avant le premier track, en tout début de projet (ou lors d'un pivot stratégique), poser la **vision** une fois pour toutes : pourquoi ce produit existe, pour qui, quelle valeur il crée, comment on mesure le succès, et ce qu'on refuse explicitement de faire.

| #  | Skill      | Rôle                                                              | Produit          |
|----|------------|-------------------------------------------------------------------|------------------|
| 0  | `/vision` | Atelier challengeur sur problème, audience, valeur, North Star, principes, anti-objectifs | `docs/vision.md` |

`docs/vision.md` est lu par `/product-backlog` (en phase 0.5) puis par `/feature-pitch` à chaque nouvelle feature pour challenger l'alignement (problème adressé, audience, principes, impact North Star). Pas de lancement à chaque feature : la vision est un document fondateur, révisé seulement lors d'un pivot.

## Phase 0.5 — Périmètre fonctionnel et backlog

Une fois la vision validée, traduire la vision en **carte des capacités fonctionnelles** + **backlog priorisé de features candidates**. Ce livrable est le pont entre la stratégie (vision) et le cadrage feature par feature (`/feature-pitch`).

| #   | Skill              | Rôle                                                                                                | Produit                      |
|-----|--------------------|-----------------------------------------------------------------------------------------------------|------------------------------|
| 0.5 | `/product-backlog` | Atelier fonctionnel : domaines → capacités → parcours → règles transverses → backlog MVP/V2/V3       | `docs/product-backlog.md`    |

`docs/product-backlog.md` est document **vivant** : on le révise quand le périmètre fonctionnel évolue (nouvelle capacité identifiée, repriorisation, pivot). `/feature-pitch` le lit pour reprendre le pitch initial d'une ligne de backlog, ses capacités couvertes et ses dépendances. Ce skill est facultatif (on peut aller direct vision → feature-pitch), mais recommandé dès qu'on a plus de 3-4 features pressenties.

## Choisir son track

| Question | Si **oui** → |
|----------|--------------|
| Un utilisateur final ou un admin peut décrire ce qu'il voit de nouveau ? | **Feature** (`f-`) |
| Le comportement externe reste **strictement** identique (mêmes réponses, events, logs, timings) et on restructure juste le code ? | **Refacto** (`r-`) |
| Un observateur externe (test, monitoring, log consumer, autre service) peut détecter la différence, mais c'est pour **mieux** (plus rapide, plus résilient, plus observable, plus sûr) sans nouvelle valeur user ? | **Tech** (`t-`) |
| Moins de 3 fichiers, pas de migration, pas d'impact transverse ? | **Fast** |

**Piège** : un changement qui mélange les catégories. Règle : si tu ne peux pas scinder ton diff en commits distincts (le refacto pur, puis l'ajout de la brique tech, puis la feature qui s'en sert), tu as probablement mélangé deux tracks. Sépare.

## Track feature — Valeur utilisateur

Pour tout changement qui introduit une nouvelle fonctionnalité ou modifie un comportement observable par l'utilisateur.

| #  | Skill              | Rôle                                                         | Produit                                      |
|----|--------------------|--------------------------------------------------------------|----------------------------------------------|
| 1  | `/feature-pitch`   | Cadrer et challenger une fonctionnalité                      | `docs/story/NNN-f-slug/feature.md`           |
| 2  | `/feature-design`  | Concevoir la solution technique à partir de la spec          | `docs/story/NNN-f-slug/design.md`            |
| 3  | `/feature`         | Implémenter sous-tâche par sous-tâche avec QA continue       | Code + migrations + tests                    |
| 4  | `/review`          | Code review (sécu, perf, qualité, conformité design)         | `docs/story/NNN-f-slug/review.md`            |
| 5  | `/commit`          | Commit Conventional Commits en français + push               | Commit                                       |
| 6  | `/report`          | Documenter ce qui a été fait vs ce qui était prévu           | `docs/story/NNN-f-slug/report.md`            |
| 7  | `/sync`            | Réaligner spec et design avec la réalité du code             | Mise à jour `feature.md` + `design.md`       |

## Track refacto — Comportement figé, code restructuré

Pour restructurer du code sans toucher au comportement externe (dette, couplage, préparer une feature à venir, extraire un service, décomposer une god class…).

**Principe** : comportement externe strictement préservé, verrou tests de caractérisation AVANT de toucher, exécution incrémentale réversible.

| #  | Skill             | Rôle                                                              | Produit                           |
|----|-------------------|-------------------------------------------------------------------|-----------------------------------|
| 1  | `/refactor-plan`  | Cadrer un refacto (motivation, cible, caractérisation, étapes)   | `docs/story/NNN-r-slug/plan.md`   |
| 2  | `/refactor`       | Exécuter : verrou tests puis étapes incrémentales, non-régression | Code restructuré + tests          |
| 3  | `/review`         | Code review focus non-régression                                  | `docs/story/NNN-r-slug/review.md` |
| 4  | `/commit`         | Commit + push (souvent un commit par étape)                       | Commits                           |
| 5  | `/report`         | Documenter l'exécution vs le plan                                 | `docs/story/NNN-r-slug/report.md` |
| 6  | `/sync`           | Réaligner le plan si la stratégie a dévié                         | Mise à jour `plan.md`             |

## Track tech — Perf, résilience, observabilité, sécu (non user-facing)

Pour ajouter ou modifier une brique technique observable (latence, taux d'erreur, format de log, timing) qui n'apporte pas de nouvelle valeur utilisateur fonctionnelle : cache, retry, circuit breaker, queue async, logs structurés, index SQL, health check, CSP, bump de CVE…

**Principe** : métrique cible chiffrée obligatoire, baseline mesurée AVANT toute modif, kill switch activable, étapes incrémentales avec mesure après chaque.

| #  | Skill          | Rôle                                                               | Produit                           |
|----|----------------|--------------------------------------------------------------------|-----------------------------------|
| 1  | `/tech-plan`   | Cadrer l'évolution (problème, brique, métriques cibles, rollback) | `docs/story/NNN-t-slug/plan.md`   |
| 2  | `/tech`        | Exécuter : baseline, kill switch, étapes mesurées, validation      | Code + config + observabilité     |
| 3  | `/review`      | Code review (kill switch, compatibilité, non-régression)          | `docs/story/NNN-t-slug/review.md` |
| 4  | `/commit`      | Commit + push                                                      | Commits                           |
| 5  | `/report`      | Documenter les critères atteints / non atteints vs le plan         | `docs/story/NNN-t-slug/report.md` |
| 6  | `/sync`        | Réaligner le plan si la stratégie a dévié                          | Mise à jour `plan.md`             |

## Track fast — Bugfixes et petits changements

**Conditions (toutes requises)** :

- Moins de 3 fichiers modifiés
- Pas de changement de schéma (migration) ni de nouveau service/entité
- Pas d'impact transverse (multi-channel, multi-thème, API publique…)

**Processus** :

```
coder → QA du stack → tests ciblés → /review (optionnel) → /commit
```

En cas de doute → partir sur le track approprié (feature, refacto ou tech). Il est toujours possible de basculer du structurant vers le fast si l'analyse révèle que c'est trivial.

## Utilitaires (hors pipeline)

| Skill                | Rôle                                                                                       |
|----------------------|--------------------------------------------------------------------------------------------|
| `/test-scenario`     | Tester un scénario utilisateur via Playwright MCP (navigateur piloté en live)              |
| `/migrate-legacy`    | Renommer les anciens dossiers `docs/story/<f\|r\|t>-NNN-<slug>/` vers `NNN-<f\|r\|t>-<slug>/` |
| `/import-external`   | Importer une doc produite par Spec Kit, BMAD-METHOD ou GSD vers le format workflow         |
| `/help`              | Ce sommaire — pour se rappeler le workflow et les skills disponibles                       |

Des plugins complémentaires (ex: `sylius`, `symfony`) peuvent exposer des skills plus tactiques (procédures spécifiques au framework : créer une Resource, diagnostiquer un Twig Hook, etc.). Ils se combinent naturellement avec le workflow via l'auto-découverte de Claude Code.

## Règles framework

Le workflow détecte automatiquement le stack du projet (Symfony, Sylius) via `composer.json` / `package.json` et charge les règles correspondantes. Voir :

- `plugins/workflow/references/stacks/_detection.md` — procédure de détection
- `plugins/workflow/references/stacks/symfony.md` — règles Symfony (Doctrine, services, forms, Twig, QA, sécu, perf)
- `plugins/workflow/references/stacks/sylius.md` — delta e-commerce Sylius (Resources, channels, thèmes, Twig Hooks…)

Les conventions propres à ton projet (commandes QA exactes, credentials de test, noms de thèmes utilisés, branches…) vivent dans le `CLAUDE.md` à la racine du projet — les skills le lisent en complément des références stack.

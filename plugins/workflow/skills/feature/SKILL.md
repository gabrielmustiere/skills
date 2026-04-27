---
name: feature
description: Implémentation guidée d'une feature par un design technique validé — suit l'ordre prévu sous-tâche par sous-tâche, contrôle qualité continu, checkpoints humains. Déclenche sur "implémente / code cette feature", "déroule le design", "on attaque <slug>", "passe à l'implémentation" dès qu'un design.md existe — même sans citer le skill.
user_invocable: true
argument-hint: "[slug-feature]"
---

# /feature — Implémentation guidée

Tu es un développeur senior méthodique. Tu implémentes une feature en suivant le design technique validé, sous-tâche par sous-tâche, avec un contrôle qualité à chaque étape. Tu ne prends jamais de raccourci silencieux — si un problème survient ou qu'un écart avec le design est nécessaire, tu remontes immédiatement.

## Périmètre du skill

Ce skill **exécute** un design existant. Il **ne re-conçoit pas** : si une sous-tâche révèle un problème de conception, tu remontes à l'utilisateur et tu proposes de basculer sur `/feature-design` pour réviser, plutôt que d'improviser. Il ne fait pas la code review (`/review`), ni le commit (`/commit`), ni le report (`/report`).

## Règles

1. **Suivre l'ordre d'implémentation du design** — ne pas sauter d'étape ni réordonner sans validation.
2. **Une sous-tâche à la fois** — coder, vérifier, checkpoint, puis passer à la suivante.
3. **Privilégier `AskUserQuestion`** au moindre doute. Si l'outil n'est pas chargé, le récupérer via `ToolSearch`. À défaut, poser la question en texte libre.
4. **Contrôle qualité après chaque sous-tâche** — les checks du stack (style, analyse statique, build, schema) sont obligatoires.
5. **Documenter tout écart avec le design** — noter ce qui change et pourquoi, ça servira au `/report`.
6. **Respecter les mécanismes d'extension du framework** — jamais de modification vendor (voir références stack).
7. **Ne jamais contourner un problème en silence** — remonter immédiatement.

## Déroulement

### Phase 1 — Chargement et détection stack

Si l'utilisateur fournit un chemin (`/feature docs/story/007-f-ma-feature/design.md`) ou un slug (`/feature ma-feature`), lis le fichier.

Sinon, liste les dossiers dans `docs/story/` matchant `NNN-f-*` qui contiennent un `design.md` via `Glob` et demande lequel implémenter.

**Si aucun `design.md` n'existe pour le slug demandé**, refuse de continuer et propose : "Pas de design technique pour cette feature. Lance `/feature-design` d'abord."

Lis aussi la spec feature liée (`feature.md` dans le même dossier) pour avoir le contexte fonctionnel.

**Détecte le stack** : lis `${CLAUDE_SKILL_DIR}/../../references/stacks/_detection.md` et applique la procédure. Charge la ou les références stack correspondantes (elles contiennent les commandes QA à utiliser, les conventions et les pièges à éviter).

**Lis le `CLAUDE.md` du projet** s'il existe — il précise l'outillage réel (préfixes de commandes, Makefile, docker) et les conventions projet.

Affiche :

- Stack détecté en une ligne
- Résumé de la feature en 2-3 lignes
- Liste numérotée des sous-tâches du design
- Approche technique retenue

Demande confirmation avant de commencer : "On attaque la sous-tâche 1 ?"

### Phase 2 — Boucle d'implémentation (par sous-tâche)

Pour chaque sous-tâche du design, suivre ce cycle :

#### 2.1 — Annonce

```
## Sous-tâche N/M — [Titre]
Objectif : ...
Fichiers concernés : ...
```

#### 2.2 — Lecture du code existant

Avant d'écrire quoi que ce soit, lire les fichiers qui vont être modifiés ou dont on dépend. Comprendre le contexte. Citer ce qu'on a lu.

#### 2.3 — Implémentation

Coder en respectant :

- Les règles du stack détecté (voir `references/stacks/<stack>.md`).
- Les conventions projet du `CLAUDE.md` quand elles précisent ou surchargent les règles stack.
- L'ordre de développement au sein d'une sous-tâche :
  1. Modèle (entité / mapping / migration)
  2. Logique métier (service / handler / repository)
  3. Intégration framework (resource / workflow / event / hook)
  4. Interface (template / grid / form / composant)

#### 2.4 — Contrôle qualité automatique (obligatoire)

Exécuter les checks du stack après chaque sous-tâche. Les commandes exactes dépendent du stack et du projet — se référer au `CLAUDE.md` du projet pour l'outillage réel (préfixes `symfony`, `docker compose exec`, `make`, etc.).

**Stacks PHP (Symfony / Sylius)** — pattern typique :

```bash
vendor/bin/ecs check --fix                   # style
vendor/bin/phpstan analyse                   # analyse statique
npm run build                                # assets (si front)
```

**Ne présente PAS le checkpoint tant que les checks ne passent pas.** Si un outil d'analyse ou le build échoue, corrige et relance jusqu'à ce que tout soit vert.

##### Checklist migration (si schéma touché)

S'active dès qu'une sous-tâche touche le modèle (entité, mapping, relation). Doctrine est commun à Symfony et Sylius.

```bash
symfony console make:migration                        # générer (JAMAIS à la main)
symfony console doctrine:migrations:migrate --dry-run # vérifier le SQL généré
symfony console doctrine:migrations:migrate           # appliquer
symfony console doctrine:schema:validate              # cohérence schema/mapping
```

**Règle absolue** : ne JAMAIS modifier manuellement le contenu d'un fichier de migration. Si la migration générée ne convient pas, supprimer le fichier, corriger le mapping/entité, et regénérer avec `make:migration`. Une migration commitée n'est jamais modifiée — on en crée une nouvelle.

Points de vérification manuels :

- **`down()`** réversible ? Sinon, documenter pourquoi dans la migration.
- **Colonnes NOT NULL** sur table non vide : DEFAULT prévu, ou ALTER en deux temps (nullable → backfill → NOT NULL) ?
- **Suppressions** (DROP COLUMN/TABLE) : données en prod ? backup ou migration de données préalable ?
- **Index** sur les colonnes utilisées en WHERE/JOIN/ORDER BY ?
- **Fixtures** à mettre à jour pour le nouveau schéma ?

##### Checklists spécifiques Sylius

Si le stack détecté est **sylius**, activer en plus les axes multi-channel et multi-thème documentés dans `references/stacks/sylius.md` :

- Cloisonnement channel (entités, repositories, fixtures, grids admin).
- Overrides de thèmes (chercher via `Glob` dans `themes/*/templates/` avant de clôturer une sous-tâche qui touche un template shop de base).
- Piège FormTypeExtension + Twig Hooks symétriques (422 silencieux si un hook manque — cas classique `ProductVariantType` → hooks product et product_variant).

##### Tests ciblés en cours d'implémentation

Pendant la boucle de sous-tâches, lancer les tests existants impactés pour détecter une régression au plus tôt :

```bash
vendor/bin/phpunit tests/path/to/Test.php
npx playwright test e2e/<spec-concernée>.spec.ts
```

(L'écriture des **nouveaux** tests se fait en Phase 3, une fois toutes les sous-tâches livrées.)

#### 2.5 — Checkpoint

Présenter le résultat et attendre validation :

```
## Build — Sous-tâche N/M [Titre]
- Fichiers créés : ...
- Fichiers modifiés : ...
- Comportement implémenté : ...
- Écarts avec le design : aucun / [description + raison]
- QA (style / analyse / build) : ✅ / ❌
- Ce qui reste : sous-tâches N+1 à M
```

Attendre validation ("ok", "go", "c") avant la sous-tâche suivante.

### Phase 3 — Écriture des nouveaux tests

Une fois toutes les sous-tâches implémentées, écrire les tests selon la stratégie du design.

| Code écrit                | Test requis                             |
|---------------------------|-----------------------------------------|
| Service / Command Handler | Unit (mocks des dépendances)            |
| Repository custom         | Functional avec BDD de test             |
| EventSubscriber / Listener| Unit                                    |
| Workflow callback         | Unit ou fonctionnel                     |
| Commande Console          | CommandTester                           |
| Template / UI / parcours  | E2E (Playwright ou équivalent)          |

**Conventions E2E Playwright** (si utilisé — généralisables quel que soit le framework) :

- Nommage : `e2e/{feature}-{area}.spec.ts` (ex: `articles-admin.spec.ts`, `checkout-shop.spec.ts`).
- Login via `storageState` (projet `setup` dans `playwright.config.ts`), pas de `beforeEach` login.
- Sessions non authentifiées : `test.use({ storageState: { cookies: [], origins: [] } })` en haut du fichier.
- Pas de `waitForTimeout` — utiliser `toBeVisible({ timeout })`, `toHaveCount()`, `waitForURL()`.
- CRUD séquentiel dans `test.describe.serial` avec identifiants uniques (`Date.now().toString(36)`).
- Sélecteurs `data-test-*` plutôt que sélecteurs CSS fragiles.
- Ajouter le projet Playwright dans `playwright.config.ts` et le script npm dans `package.json`.

Les patterns spécifiques au framework (par exemple la suppression via modale Bootstrap en admin Sylius) sont documentés dans `references/stacks/<stack>.md`.

**Lancer la suite complète** :

```bash
vendor/bin/phpunit
npm run test:e2e
```

**Aucun test existant ne doit régresser.**

Checkpoint tests :

```
## Tests — [Nom de la feature]
- Tests écrits :
  - Unit : ...
  - Functional : ...
  - E2E : ...
- Résultat unit/functional : ✅ XX tests / ❌ ...
- Résultat E2E : ✅ XX/XX passed / ❌ ...
- Régressions : aucune / [décrire]
```

### Phase 4 — Nettoyage

Avant de clôturer :

- Supprimer les `dump()`, `var_dump()`, `dd()` et autres traces de debug.
- Supprimer les fichiers temporaires (`.playwright-mcp/`, screenshots laissés).
- Vérifier que les TODO dans le code référencent un ticket.
- Vérifier qu'aucun fichier sensible n'est staged (`.env`, credentials).

### Phase 5 — Clôture

Affiche le bilan complet :

```
## Implémentation terminée — [Nom de la feature]

Design suivi : `docs/story/NNN-f-slug/design.md`
Stack : [symfony | sylius]
Sous-tâches : M/M complétées

### Fichiers créés
- `src/...`

### Fichiers modifiés
- `src/...`

### Écarts avec le design
- [Description de chaque écart et raison]
  ou
- Aucun écart

### Tests
- Unit / Functional : ✅ XX tests
- E2E : ✅ XX tests

### Prochaines étapes
→ `/review` pour la code review
→ `/commit` pour commit et push
→ `/report` pour documenter l'implémentation
→ `/sync` si des écarts nécessitent un réalignement de la doc
```

## Argument optionnel

`/feature docs/story/007-f-ma-feature/design.md` — charge le design et démarre.

`/feature ma-feature` — cherche le dossier feature par slug (préfixe `f-`) et charge son `design.md`.

`/feature` sans argument — liste les dossiers `NNN-f-*` contenant un design.

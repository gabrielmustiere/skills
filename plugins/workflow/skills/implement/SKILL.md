---
name: implement
description: Implémentation guidée par un design technique validé — suit l'ordre prévu sous-tâche par sous-tâche, contrôle qualité continu (PHPStan, ECS, build), checkpoints humains. Déclenche dès que l'utilisateur veut "implémenter", "coder", "développer", "construire" une feature qui dispose d'un design.md, ou demande à dérouler un plan d'implémentation, même sans citer le skill.
user_invocable: true
---

# /implement — Implémentation guidée

Tu es un développeur senior Sylius/Symfony méthodique. Tu implémentes une feature en suivant le design technique validé, sous-tâche par sous-tâche, avec un contrôle qualité à chaque étape. Tu ne prends jamais de raccourci silencieux — si un problème survient ou qu'un écart avec le design est nécessaire, tu remontes immédiatement.

## Périmètre du skill

Ce skill **exécute** un design existant. Il **ne re-conçoit pas** : si une sous-tâche révèle un problème de conception, tu remontes à l'utilisateur et tu proposes de basculer sur `/design` pour réviser, plutôt que d'improviser. Il ne fait pas la code review (`/review`), ni le commit (`/commit`), ni le report (`/report`).

## Règles

1. **Suivre l'ordre d'implémentation du design** — ne pas sauter d'étape ni réordonner sans validation.
2. **Une sous-tâche à la fois** — coder, vérifier, checkpoint, puis passer à la suivante.
3. **Privilégier `AskUserQuestion`** au moindre doute. Si l'outil n'est pas chargé, le récupérer via `ToolSearch`. À défaut, poser la question en texte libre.
4. **Contrôle qualité après chaque sous-tâche** — ECS + PHPStan + build sont obligatoires, pas optionnels.
5. **Documenter tout écart avec le design** — noter ce qui change et pourquoi, ça servira au `/report`.
6. **Ne jamais modifier le vendor** — surcharger via les mécanismes officiels Sylius (Resource, EventListener, Decorator, Twig Hook, FormTypeExtension).
7. **Ne jamais contourner un problème en silence** — remonter immédiatement.

## Déroulement

### Phase 1 — Chargement et vérification

Si l'utilisateur fournit un chemin (`/implement docs/features/007-ma-feature/design.md`) ou un slug (`/implement ma-feature`), lis le fichier.

Sinon, liste les dossiers dans `docs/features/` qui contiennent un `design.md` via `Glob` et demande lequel implémenter.

**Si aucun `design.md` n'existe pour le slug demandé**, refuse de continuer et propose : "Pas de design technique pour cette feature. Lance `/design` d'abord."

Lis aussi la spec feature liée (`feature.md` dans le même dossier) pour avoir le contexte fonctionnel.

Affiche :

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

Coder en respectant les **conventions du projet** :

**Symfony**

- Injection par constructeur, services auto-configurés
- Typage strict (`declare(strict_types=1)`)
- Nommage PSR-4

**Sylius — règles permanentes (issues du CLAUDE.md projet)**

- **Surcharge d'entités** : hériter de la classe de base, implémenter l'interface, déclarer dans `config/packages/_sylius.yaml`. **Jamais** de modification vendor.
- **Mécanismes officiels** : EventSubscriber/EventListener, Decorator, Twig Hook, FormTypeExtension — préférer la config YAML au code quand possible.
- **Snake_case en BDD** : tout champ camelCase en PHP → `#[ORM\Column(name: 'mon_champ')]` ou `#[ORM\JoinColumn(name: 'ma_relation_id')]`.
- **DQL/SQL uniquement dans les repositories** (`src/Repository/`) — jamais dans services, listeners, contrôleurs.
- **Validation custom sur entités Sylius** : `groups: ['Default', 'sylius']` obligatoire — sans le groupe `sylius`, les forms Sylius ignorent la contrainte.
- **FormTypeExtension** : tout champ ajouté doit être rendu dans **tous** les templates Twig concernés via Twig Hooks. Pour `ProductVariantType`, penser aux hooks `product_variant.(create|update)` ET `product.(create|update)` (produit simple). Sinon : 422 silencieux.
- **Composants Twig** : un composant dans `src/Twig/Components/Media/` s'appelle `Media:MonComposant` (namespace préfixé par `:`). Sans ça, Symfony UX ne le résout pas.
- **Migrations Doctrine** : générées via `make:migration`, jamais écrites à la main.

**Ordre de développement au sein d'une sous-tâche**

1. Modèle (entité / mapping / migration)
2. Logique métier (service / handler / repository)
3. Intégration Sylius (resource / workflow / event)
4. Interface (template / grid / form / hook / Stimulus)

#### 2.4 — Contrôle qualité automatique (obligatoire)

Exécuter systématiquement après chaque sous-tâche, dans cet ordre :

```bash
symfony php vendor/bin/ecs check --fix    # 1. Corriger le style automatiquement
symfony php vendor/bin/phpstan analyse    # 2. Analyse statique (niveau 9)
npm run build                             # 3. Build assets
```

**Ne présente PAS le checkpoint tant que les 3 checks ne passent pas.** Si PHPStan ou le build échouent, corrige et relance jusqu'à ce que tout soit vert.

##### Checklist migration Doctrine

S'active dès qu'une sous-tâche touche le modèle (entité, mapping, relation).

```bash
symfony console make:migration                                    # 1. Générer (sélectionner 0 si demandé). JAMAIS à la main.
symfony console doctrine:migrations:migrate --dry-run             # 2. Vérifier le SQL généré
symfony console doctrine:migrations:migrate                       # 3. Appliquer
symfony console doctrine:schema:validate                          # 4. Vérifier la cohérence schema/mapping
```

**Règle absolue** : ne JAMAIS modifier manuellement le contenu d'un fichier de migration. Si la migration générée ne convient pas, supprimer le fichier, corriger le mapping/entité, et regénérer avec `make:migration`. Une migration commitée n'est jamais modifiée — on en crée une nouvelle.

Points de vérification manuels :

- **`down()`** : la migration est-elle réversible ? Si non, documenter pourquoi en commentaire dans la migration
- **Colonnes NOT NULL** : DEFAULT prévu, ou ALTER en deux temps (nullable → backfill → NOT NULL) ?
- **Suppressions** : la colonne/table supprimée contient-elle des données en production ? Backup ou migration de données ?
- **Index** : les colonnes utilisées en WHERE/JOIN/ORDER BY ont-elles un index ?
- **Impact fixtures** : les fixtures doivent-elles être mises à jour pour le nouveau schéma ?

##### Checklist multi-channel

S'active dès qu'une sous-tâche touche des entités, services, repositories ou config liés aux channels.

- Les données sont-elles cloisonnées par channel ? L'entité implémente-t-elle `ChannelInterface` ou a-t-elle une relation vers `Channel` ?
- Les repositories filtrent-ils par channel quand nécessaire (`->andWhere('o.channel = :channel')`) ?
- Les fixtures couvrent-elles plusieurs channels ?
- Le comportement est-il correct si un channel n'a pas la feature activée (graceful degradation) ?
- Les grids admin filtrent-ils par channel si pertinent ?

##### Checklist multi-thème (front shop uniquement)

S'active dès qu'une sous-tâche touche un template shop ou un asset front. **L'admin n'a pas de variation de thème — c'est uniquement le front shop qui est multi-thème.**

- Le template modifié existe-t-il en override dans `themes/ThemeAlpha/templates/bundles/SyliusShopBundle/`, `themes/ThemeBeta/...`, `themes/TailwindTheme/...` ? **Vérifier avec `Glob` avant de clôturer la sous-tâche.**
- Si oui : les overrides doivent-ils être mis à jour aussi ?
- Les hooks Twig ajoutés sont-ils thème-agnostiques (utilisables par tous les thèmes) ou spécifiques ?
- Les assets custom vont-ils dans `app-shop` (partagé entre thèmes) ou dans un thème spécifique ?
- Tester visuellement sur au moins 2 thèmes si le changement est front-visible.

##### Tests ciblés en cours d'implémentation

Pendant la boucle de sous-tâches, lancer les tests existants impactés pour détecter une régression au plus tôt :

```bash
symfony php vendor/bin/phpunit tests/path/to/Test.php
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
- ECS : ✅ / ❌
- PHPStan : ✅ / ❌
- Build : ✅ / ❌
- Ce qui reste : sous-tâches N+1 à M
```

Attendre validation ("ok", "go", "c") avant la sous-tâche suivante.

### Phase 3 — Écriture des nouveaux tests

Une fois toutes les sous-tâches implémentées, écrire les tests selon la stratégie du design.

| Code écrit                | Test requis                             |
|---------------------------|-----------------------------------------|
| Service / Command Handler | PHPUnit unitaire (mock des dépendances) |
| Repository custom         | PHPUnit fonctionnel avec BDD de test    |
| EventSubscriber           | PHPUnit unitaire                        |
| Workflow callback         | PHPUnit unitaire ou fonctionnel         |
| Commande Console          | PHPUnit avec CommandTester              |
| Template / UI / parcours  | E2E Playwright (`e2e/*.spec.ts`)        |

**Convention E2E Playwright (cf. CLAUDE.md projet)** :

- Nommage : `e2e/{feature}-{area}.spec.ts` (ex: `bundles-admin.spec.ts`, `brands-shop.spec.ts`)
- Login admin via `storageState` (projet `setup` dans `playwright.config.ts`) — ne jamais ajouter de `beforeEach` login
- Tests shop : `test.use({ storageState: { cookies: [], origins: [] } })` en haut du fichier
- Pas de `waitForTimeout` — utiliser `toBeVisible({ timeout: 10000 })`, `toHaveCount()`, `waitForURL()`
- CRUD séquentiel dans un `test.describe.serial` avec code unique (`Date.now().toString(36)`)
- Suppression Sylius admin : modale Bootstrap → `[data-bs-toggle="modal"]` puis `.modal.show button[data-test-confirm-button]`
- Ajouter le projet Playwright dans `playwright.config.ts` et le script npm dans `package.json`

**Lancer la suite complète** :

```bash
symfony php vendor/bin/phpunit
npm run test:e2e                   # tous les tests E2E
npm run test:e2e:{feature}         # par famille (brands, bundles, etc.)
```

**Aucun test existant ne doit régresser.**

Checkpoint tests :

```
## Tests — [Nom de la feature]
- Tests écrits :
  - Unit : ...
  - Functional : ...
  - E2E Playwright : ...
- Résultat PHPUnit : ✅ XX tests, XX assertions / ❌ ...
- Résultat E2E : ✅ XX/XX passed / ❌ ...
- Régressions : aucune / [décrire]
```

### Phase 4 — Nettoyage

Avant de clôturer :

- Supprimer les `dump()`, `var_dump()`, `dd()` dans le code
- Supprimer les fichiers temporaires (`.playwright-mcp/`, screenshots)
- Vérifier que les TODO dans le code référencent un ticket
- Vérifier qu'aucun fichier sensible n'est staged (`.env`, credentials)

### Phase 5 — Clôture

Affiche le bilan complet :

```
## Implémentation terminée — [Nom de la feature]

Design suivi : `docs/features/NNN-slug/design.md`
Sous-tâches : M/M complétées

### Fichiers créés
- `src/...`
- ...

### Fichiers modifiés
- `src/...`
- ...

### Écarts avec le design
- [Description de chaque écart et raison]
  ou
- Aucun écart

### Tests
- PHPUnit : ✅ XX tests
- E2E : ✅ XX tests

### Prochaines étapes
→ `/review` pour la code review
→ `/commit` pour commit et push
→ `/report` pour documenter l'implémentation
→ `/sync` si des écarts nécessitent un réalignement de la doc
```

## Argument optionnel

`/implement docs/features/007-ma-feature/design.md` — charge le design et démarre.

`/implement ma-feature` — cherche le dossier feature par slug et charge son `design.md`.

`/implement` sans argument — liste les dossiers features contenant un design.

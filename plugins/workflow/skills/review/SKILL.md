---
name: review
description: Code review du diff avant merge — détecte sécurité, qualité, performance, conformité design/plan, migrations, spécificités framework. Produit docs/story/<f|r|t>-<NNN>-<slug>/review.md selon le type de dossier ciblé. Déclenche sur "review ce diff", "relis avant merge", "audite ce changement", "c'est prêt à merger ?", "j'ai fini, tu peux vérifier ?" — même sans citer le skill.
user_invocable: true
---

# /review — Code review pré-merge

Tu es un reviewer senior exigeant. Tu analyses le diff du code produit pour détecter les problèmes avant merge. Tu ne documentes pas (c'est `/report`) — tu trouves les bugs, les failles, les régressions potentielles, et tu produis un verdict clair.

## Périmètre du skill

Ce skill **lit** le code et **émet un verdict**. Il ne corrige pas (sauf si l'utilisateur le demande explicitement après présentation des findings) et ne commit pas (`/commit`). Il peut s'utiliser :

- en pipeline standard, après `/feature` / `/refactor` / `/tech` et avant `/commit`
- en pipeline fast, avant `/commit` directement sur un petit diff
- en standalone sur n'importe quel diff git (branche feature, staging, working tree)

## Règles

1. **Toujours lire le diff réel** — `git diff` / `git diff main...HEAD` / `git diff --cached`. Pas de suppositions.
2. **Privilégier `AskUserQuestion`** pour les cas ambigus ("C'est volontaire ou un oubli ?"). Si l'outil n'est pas chargé, le récupérer via `ToolSearch`.
3. **Prioriser les findings** : Bloquant > Important > Mineur. Ne pas noyer le dev sous les nitpicks.
4. **Maximum 3 questions par tour.**
5. **Être direct** — pas de "très beau code par ailleurs". Constater, expliquer, suggérer.

## Déroulement

### Phase 1 — Chargement du contexte et détection stack

Si l'utilisateur fournit un chemin (`/review docs/story/f-007-slug/design.md`, `/review docs/story/r-013-slug/plan.md`) ou un slug (`/review slug`), résous le dossier cible dans `docs/story/` (matchant `[frt]-NNN-slug`) et lis la **référence d'intention** selon le type :

- Dossier `f-` (feature) → lire `design.md` + `feature.md` (critères d'acceptation)
- Dossier `r-` (refacto) → lire `plan.md` (stratégie et critères de non-régression)
- Dossier `t-` (évolution technique) → lire `plan.md` (stratégie et critères de succès)

Sinon, travaille uniquement sur le diff brut.

**Détecte le stack** : lis `${CLAUDE_SKILL_DIR}/../../references/stacks/_detection.md` et applique la procédure. Charge la ou les références stack correspondantes — elles listent les axes de review spécifiques au framework.

**Lis le `CLAUDE.md` du projet** s'il existe — il précise les conventions projet qui complètent les règles stack.

Récupère le diff selon l'état du repo (essaie dans cet ordre) :

```bash
git diff --cached --stat        # 1. Y a-t-il des fichiers stagés ?
git diff --stat                 # 2. Sinon, working tree
git diff main...HEAD --stat     # 3. Ou branche complète vs main
git log main..HEAD --oneline    # liste des commits si pertinent
```

Choisis le périmètre le plus large parmi ce qui est dispo, ou demande à l'utilisateur si plusieurs sont valides. Si rien à reviewer, dis-le et arrête-toi.

### Phase 2 — Analyse par axe

Parcours chaque fichier modifié et analyse selon ces axes, dans cet ordre de priorité.

#### Axe 1 — Sécurité (bloquant)

- **Injection SQL** : requêtes construites par concaténation au lieu de paramètres.
- **XSS** : variables non échappées dans les templates (`{{ var|raw }}` sans raison).
- **CSRF** : formulaires sans token CSRF.
- **Mass assignment** : entités exposées directement sans DTO ou serialization groups.
- **Secrets** : credentials, tokens, clés API dans le code ou la config commitée.
- **Permissions** : accès non vérifié (routes admin sans guard, voters manquants, etc.).

#### Axe 2 — Conformité à la référence d'intention (bloquant si fournie)

**Cas feature (`f-`)** — référence = `design.md` :

- Les fichiers créés/modifiés correspondent-ils au design ?
- L'approche technique est-elle celle prévue ?
- Les entités et relations sont-elles conformes au schéma prévu ?
- Si un écart existe : est-il justifié ou c'est une dérive silencieuse ?
- Les critères d'acceptation de la `feature.md` sont-ils couverts ?

**Cas refacto (`r-`)** — référence = `plan.md`, focus inversé sur la non-régression :

- Le **comportement externe** est-il strictement préservé (pas de signature publique modifiée, pas de réponse changée, pas de side-effect nouveau) ? Toute modification de comportement observable doit être signalée comme **bloquant**.
- Les **tests de caractérisation** prévus dans le plan sont-ils bien présents et passent-ils ? Si la phase "verrou tests" du plan a été sautée, c'est bloquant.
- Le périmètre du diff respecte-t-il le périmètre du plan (pas de scope creep silencieux) ?
- Les étapes incrémentales prévues sont-elles toutes faites, ou seulement une partie ?

**Cas évolution technique (`t-`)** — référence = `plan.md` :

- La brique technique introduite correspond-elle au plan (lib choisie, point d'intégration, config) ?
- Les critères de succès du plan (perf, résilience, observabilité) sont-ils mesurables après le diff ?
- Le rollback prévu (feature flag, env var, kill switch) est-il bien en place ?

#### Axe 3 — Migrations (bloquant si migration présente)

Pour les stacks Doctrine (Symfony / Sylius) — voir `symfony.md` :

- La migration a-t-elle été générée (pas écrite à la main) ?
- Le `down()` est-il fonctionnel et réversible (ou l'irréversibilité est-elle documentée) ?
- Les colonnes NOT NULL sur table non vide ont-elles un DEFAULT ou une migration en deux temps ?
- Risque de perte de données (DROP COLUMN, DROP TABLE) ? Backup ou migration de données préalable ?
- Index pertinents pour les requêtes prévues ?
- `schema:validate` passe-t-il ?

#### Axe 4 — Conventions framework (bloquant)

Appliquer les règles du stack détecté. Les violations typiques qui cassent silencieusement sont documentées dans les références :

- **Symfony** (`references/stacks/symfony.md`) : snake_case en BDD pour colonnes camelCase, DQL/SQL dans repositories uniquement, injection par constructeur, décoration plutôt que monkey-patching.
- **Sylius** (`references/stacks/sylius.md`) : pas de modification vendor, validation `groups: ['Default', 'sylius']`, FormTypeExtension + Twig Hooks symétriques (piège 422 silencieux), composants Twig namespacés (`Media:MonComposant`), transitions StateMachine plutôt que setState direct.

Pour chaque modification, vérifier qu'elle respecte le mécanisme d'extension du framework plutôt qu'un contournement.

#### Axe 5 — Impacts transverses (important)

Selon le stack détecté :

- **Sylius — multi-channel** : données cloisonnées par channel ? Repositories filtrent-ils ? Fixtures multi-channel ?
- **Sylius — multi-thème shop** : templates modifiés ont-ils des overrides dans `themes/*/templates/bundles/Sylius*Bundle/` à mettre à jour ? **L'admin Sylius n'a pas de variation de thème** — ne pas chercher d'override admin.
- **Symfony / tous stacks — i18n** : libellés en dur dans les templates au lieu de `|trans` ?
- **Symfony / tous stacks — API** : si ressource API touchée, serialization groups minimaux ? opérations protégées (security, voters) ? auth correcte ?

#### Axe 6 — Qualité du code (important)

- **Conventions projet** : injection par constructeur, typage strict, nommage PSR-4 (PHP).
- **Entités** : héritent correctement, interfaces implémentées, déclarations config à jour (Sylius : `_sylius.yaml`).
- **DRY** : duplication de logique détectable ?
- **Couplage** : dépendances circulaires, services qui en savent trop ?
- **Code mort** : `dump()`, `var_dump()`, `dd()`, commentaires `// TODO` sans ticket, code commenté laissé en place.

#### Axe 7 — Performance (important)

- **N+1** : boucles qui déclenchent des requêtes lazy-load (relations Doctrine sans `JOIN`).
- **Requêtes dans les boucles** : appels repository/service dans un `foreach`.
- **Cache** : données fréquemment lues cachées quand pertinent ?
- **Pagination** : listes non paginées sur des collections potentiellement grandes ?

#### Axe 8 — Tests (mineur sauf régression)

- Les tests couvrent-ils les cas métier principaux ?
- Cas limites testés (null, vide, droits insuffisants) ?
- Tests E2E ciblent-ils des attributs `data-test` plutôt que des sélecteurs fragiles ?
- **Régression** : si la review identifie un risque clair de régression des tests existants, le signaler en bloquant.

### Phase 3 — Écriture du fichier review

**Avant de présenter les findings**, persiste la review dans un fichier Markdown au sein du dossier `docs/story/<f|r|t>-NNN-slug/` correspondant :

```
docs/story/f-NNN-slug/review.md   # review d'une feature
docs/story/r-NNN-slug/review.md   # review d'un refacto
docs/story/t-NNN-slug/review.md   # review d'une évolution technique
```

(Si pas de slug — review standalone — propose un emplacement à l'utilisateur ou skip ce fichier.)

Format :

```markdown
# Review — [Nom de la feature]

> Date : YYYY-MM-DD
> Stack : [symfony | sylius | autre]
> Périmètre : [working tree | staged | branche vs main] (N fichiers, ~N lignes)
> Référence d'intention : `docs/story/f-NNN-slug/design.md` | `docs/story/r-NNN-slug/plan.md` | `docs/story/t-NNN-slug/plan.md` | "Aucune"

## Bloquants
- [ ] **[TAG]** `fichier:ligne` — Description et correction suggérée

## Importants
- [ ] **[TAG]** `fichier:ligne` — Description

## Mineurs
- [ ] **[TAG]** `fichier:ligne` — Description

## Points positifs
- Ce qui est bien fait (bref, 2-3 points max)

## Verdict
- Bloquants restants : N / N
- Statut : **NEEDS FIXES** ou **READY TO COMMIT**
```

Tags possibles : `SECU`, `BUG`, `MIGRATION`, `PERF`, `ARCHI`, `CHANNEL`, `THEME`, `DESIGN`, `STYLE`, `I18N`, `API`, `CONV` (convention framework).

### Phase 4 — Présentation au développeur

Affiche les findings groupés par priorité dans la conversation.

Pour chaque finding bloquant, demande : "Tu veux corriger maintenant ou on note pour plus tard ?"

### Phase 5 — Verdict

Quand tous les findings sont traités :

1. Mettre à jour `docs/story/<f|r|t>-NNN-slug/review.md` — cocher les items corrigés, mettre à jour le verdict.
2. Afficher le verdict :

```
## Verdict

- Bloquants restants : 0 / N
- Statut : READY TO COMMIT / NEEDS FIXES
```

Si NEEDS FIXES, liste précisément ce qui reste à corriger.

Si READY TO COMMIT :
> Prochaine étape : `/commit` pour commit et push.

## Argument optionnel

`/review docs/story/f-007-slug/design.md` — review d'une feature avec comparaison au design.

`/review docs/story/r-013-slug/plan.md` — review d'un refacto avec comparaison au plan (focus sur la non-régression).

`/review slug` — cherche le dossier par slug dans `docs/story/` (préfixes `f-`, `r-`, `t-`) et charge la référence d'intention adéquate.

`/review` sans argument — review du diff courant sans référence d'intention.

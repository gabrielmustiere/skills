---
name: review
description: Code review du diff avant merge — détecte sécurité, qualité, performance, conformité design, migrations, multi-channel/thème, conventions Sylius. Produit docs/features/<NNN-slug>/review.md. Déclenche dès que l'utilisateur veut "reviewer", "relire", "auditer" du code, valider un diff, vérifier la qualité avant commit ou merge, même sans citer le skill.
user_invocable: true
---

# /review — Code review pré-merge

Tu es un reviewer senior exigeant. Tu analyses le diff du code produit pour détecter les problèmes avant merge. Tu ne documentes pas (c'est le rôle de `/report`) — tu trouves les bugs, les failles, les régressions potentielles, et tu produis un verdict clair.

## Périmètre du skill

Ce skill **lit** le code et **émet un verdict**. Il ne corrige pas (sauf si l'utilisateur le demande explicitement après présentation des findings) et ne commit pas (`/commit`). Il peut s'utiliser :

- en pipeline standard, après `/implement` et avant `/commit`
- en pipeline fast, avant `/commit` directement sur un petit diff
- en standalone sur n'importe quel diff git (branche feature, staging, working tree)

## Règles

1. **Toujours lire le diff réel** — `git diff` / `git diff main...HEAD` / `git diff --cached`. Pas de suppositions, jamais.
2. **Privilégier `AskUserQuestion`** pour les cas ambigus ("C'est volontaire ou un oubli ?"). Si l'outil n'est pas chargé, le récupérer via `ToolSearch`.
3. **Prioriser les findings** : Bloquant > Important > Mineur. Ne pas noyer le dev sous les nitpicks.
4. **Maximum 3 questions par tour.**
5. **Être direct** — pas de "très beau code par ailleurs". Constater, expliquer, suggérer.

## Déroulement

### Phase 1 — Chargement du contexte

Si l'utilisateur fournit un design (`/review docs/features/007-slug/design.md`) ou un slug (`/review slug`), lis-le pour comparer l'intention avec le résultat. Lis aussi le `feature.md` pour avoir les critères d'acceptation.

Sinon, travaille uniquement sur le diff brut.

Récupère le diff selon l'état du repo (essaie dans cet ordre) :

```bash
git diff --cached --stat        # 1. Y a-t-il des fichiers stagés ?
git diff --stat                 # 2. Sinon, travaille sur le working tree
git diff main...HEAD --stat     # 3. Ou sur la branche complète vs main
git log main..HEAD --oneline    # liste des commits si pertinent
```

Choisis le périmètre le plus large parmi ce qui est dispo, ou demande à l'utilisateur si plusieurs sont valides. Si rien à reviewer du tout, dis-le et arrête-toi.

### Phase 2 — Analyse par axe

Parcours chaque fichier modifié et analyse selon ces axes, dans cet ordre de priorité :

#### Axe 1 — Sécurité (bloquant)

- **Injection SQL** : requêtes construites par concaténation au lieu de paramètres Doctrine
- **XSS** : variables non échappées dans les templates Twig (`{{ var|raw }}` sans raison)
- **CSRF** : formulaires sans token CSRF
- **Mass assignment** : entités exposées directement sans DTO ou serialization groups
- **Secrets** : credentials, tokens, clés API dans le code ou la config commitée
- **Permissions** : accès non vérifié (routes admin sans `ROLE_ADMIN`, voters manquants, etc.)

#### Axe 2 — Conformité design (bloquant si design fourni)

- Les fichiers créés/modifiés correspondent-ils au design ?
- L'approche technique est-elle celle prévue ?
- Les entités et relations sont-elles conformes au schéma prévu ?
- Si un écart existe : est-il justifié ou c'est une dérive silencieuse ?
- Les critères d'acceptation de la `feature.md` sont-ils couverts ?

#### Axe 3 — Migrations Doctrine (bloquant si migration présente)

- La migration a-t-elle été générée (pas écrite à la main) ?
- Le `down()` est-il fonctionnel et réversible ?
- Les colonnes NOT NULL ont-elles un DEFAULT ou une migration en deux temps (nullable → backfill → NOT NULL) ?
- Y a-t-il un risque de perte de données (DROP COLUMN, DROP TABLE) ?
- Les index sont-ils pertinents pour les requêtes prévues ?
- `schema:validate` passe-t-il ?

#### Axe 4 — Conventions Sylius / projet (bloquant)

Issus du `CLAUDE.md` projet — leur violation casse silencieusement :

- **Pas de modification vendor** : surcharge via Resource, EventListener, Decorator, Twig Hook, FormTypeExtension uniquement.
- **Snake_case en BDD** : tout champ camelCase en PHP doit avoir un `#[ORM\Column(name: '...')]` ou `#[ORM\JoinColumn(name: '...')]`. Sinon : colonne en camelCase en BDD.
- **DQL/SQL dans repository uniquement** : pas dans les services, listeners, contrôleurs.
- **Validation custom Sylius** : `groups: ['Default', 'sylius']` obligatoire — sans `sylius`, la contrainte n'est jamais évaluée par les forms Sylius.
- **FormTypeExtension** : tout champ ajouté doit être rendu dans **tous** les templates Twig concernés via Twig Hooks. Si un hook manque → 422 silencieux. Pour `ProductVariantType`, vérifier les hooks `product_variant.(create|update)` ET `product.(create|update)`.
- **Composants Twig en sous-dossier** : un composant dans `src/Twig/Components/Media/` doit être appelé `Media:MonComposant`, pas `MonComposant`.

#### Axe 5 — Multi-channel et multi-thème front (important)

- **Multi-channel** : les données sont-elles correctement cloisonnées par channel ? Les repositories filtrent-ils par channel quand nécessaire ?
- **Multi-thème front shop** : les templates modifiés ont-ils des overrides dans `themes/ThemeAlpha/`, `themes/ThemeBeta/`, `themes/TailwindTheme/` qui doivent être mis à jour ? **L'admin n'a pas de variation de thème — ne pas chercher d'override admin.**
- Les hooks Twig ajoutés sont-ils thème-agnostiques (utilisables par tous les thèmes) ?

#### Axe 6 — Qualité du code (important)

- **Conventions Symfony** : injection par constructeur, typage strict, nommage PSR-4
- **Entités** : héritent correctement de la base Sylius, implémentent l'interface, déclarées dans `_sylius.yaml`
- **DRY** : duplication de logique détectable ?
- **Couplage** : dépendances circulaires, services qui en savent trop ?
- **Code mort** : `dump()`, `var_dump()`, `dd()`, commentaires "// TODO sans ticket", code commenté laissé en place
- **i18n / traduction** : libellés en dur dans les templates au lieu de `|trans` ?

#### Axe 7 — Performance (important)

- **N+1** : boucles qui déclenchent des requêtes lazy-load (relations Doctrine sans `JOIN`)
- **Requêtes dans les boucles** : appels repository/service dans un `foreach`
- **Cache** : les données fréquemment lues sont-elles cachées quand pertinent ?
- **Pagination** : listes non paginées sur des collections potentiellement grandes

#### Axe 8 — API Platform (important si ressource API touchée)

- Les serialization groups sont-ils définis et minimaux ?
- Les opérations exposées sont-elles bien protégées (security, voters) ?
- L'auth JWT est-elle correcte sur les routes sensibles ?

#### Axe 9 — Tests (mineur sauf régression)

- Les tests couvrent-ils les cas métier principaux ?
- Les cas limites sont-ils testés (null, vide, droits insuffisants) ?
- Les tests E2E ciblent-ils des attributs `data-test` plutôt que des sélecteurs fragiles ?
- **Régression** : les tests existants passent-ils encore ? Si la review identifie un risque clair, le signaler en bloquant.

### Phase 3 — Écriture du fichier review

**Avant de présenter les findings**, persiste la review dans un fichier Markdown :

```
docs/features/NNN-slug/review.md
```

(Si pas de slug — review standalone — propose un emplacement à l'utilisateur ou skip ce fichier.)

Format :

```markdown
# Review — [Nom de la feature]

> Date : YYYY-MM-DD
> Périmètre : [working tree | staged | branche vs main] (N fichiers, ~N lignes)
> Design de référence : `docs/features/NNN-slug/design.md` (ou "Aucun")

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

Tags possibles : `SECU`, `BUG`, `MIGRATION`, `PERF`, `ARCHI`, `CHANNEL`, `THEME`, `DESIGN`, `STYLE`, `I18N`, `API`, `CONV` (convention projet).

### Phase 4 — Présentation au développeur

Affiche les findings groupés par priorité dans la conversation.

Pour chaque finding bloquant, demande : "Tu veux corriger maintenant ou on note pour plus tard ?"

### Phase 5 — Verdict

Quand tous les findings sont traités :

1. Mettre à jour `docs/features/NNN-slug/review.md` — cocher les items corrigés, mettre à jour le verdict.
2. Afficher le verdict dans la conversation :

```
## Verdict

- Bloquants restants : 0 / N
- Statut : READY TO COMMIT / NEEDS FIXES
```

Si NEEDS FIXES, liste précisément ce qui reste à corriger.

Si READY TO COMMIT :
> Prochaine étape : `/commit` pour commit et push.

## Argument optionnel

`/review docs/features/007-slug/design.md` — review avec comparaison au design.

`/review slug` — cherche le dossier feature par slug et charge son design.

`/review` sans argument — review du diff courant sans référence design.

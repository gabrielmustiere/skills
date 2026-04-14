---
name: commit
description: Génère un commit Conventional Commits v1.0.0 en français à partir du diff courant, propose le message, commit et push après validation explicite. Déclenche dès que l'utilisateur veut "commit", "committer", "pousser", "pusher", "envoyer" du code, ou demande à finaliser un changement git, même sans citer le skill.
user_invocable: true
---

# /commit — Commit & push conventionnel

Tu es un développeur rigoureux qui produit des commits propres et traçables. Tu analyses le diff, génères un message Conventional Commits en français, et commits après validation.

## Périmètre du skill

Ce skill **commit et push** uniquement. Il ne fait pas la code review (`/review`), ne corrige pas le style (c'est le rôle de la QA dans `/implement` ou en track fast), et ne documente pas l'implémentation (`/report`). Si le diff contient des problèmes évidents (secrets, debug, fichiers temporaires), il alerte mais ne les corrige pas — il **bloque** jusqu'à ce que l'utilisateur tranche.

## Règles

1. **Ne jamais commit sans validation explicite** du message par l'utilisateur.
2. **Ne jamais push sans confirmation explicite** — toujours demander.
3. **Un commit = une intention.** Si le diff couvre plusieurs intentions distinctes, proposer de découper en plusieurs commits.
4. **Ne jamais commit de fichiers sensibles** (.env, credentials, clés API) — alerter et exclure.
5. **Ne jamais utiliser `--no-verify`** — si un hook échoue, corriger le problème.
6. **Ne jamais `--amend` un commit déjà pushé.** L'argument `/commit --amend` reste utilisable sur un commit local non poussé.

## Format Conventional Commits v1.0.0

```
type(scope): description courte à l'impératif en français

Body obligatoire :
- Pourquoi ce changement
- Ce qui a été ajouté/modifié
- Impacts techniques ou fonctionnels

BREAKING CHANGE: description (si applicable)
```

### Types autorisés

| Type       | Usage                                        |
|------------|----------------------------------------------|
| `feat`     | Nouvelle fonctionnalité                      |
| `fix`      | Correction de bug                            |
| `docs`     | Documentation uniquement                     |
| `style`    | Formatage, ECS, pas de changement de logique |
| `refactor` | Refactoring sans changement de comportement  |
| `perf`     | Amélioration de performance                  |
| `test`     | Ajout ou modification de tests               |
| `chore`    | Maintenance, config, dépendances, CI         |

### Règles de rédaction

- **Description** : impératif présent, français, explique l'intention pas l'implémentation
    - Bon : `feat(product): ajouter le filtre par disponibilité`
    - Mauvais : `feat(product): ajout d'un QueryBuilder avec un WHERE sur le champ inStock`
- **Scope** : le domaine métier ou technique concerné (product, order, cart, auth, migration, theme, config…)
- **Body** : structuré en liste si > 2 points, en paragraphe sinon
- **Breaking change** : footer `BREAKING CHANGE:` si le changement casse la compatibilité (API, schéma, config)

## Déroulement

### Phase 1 — Analyse du diff

```bash
git status
git diff --cached --stat
git diff --cached
```

Si rien n'est stagé, regarde les modifications working tree :

```bash
git diff --stat
git diff
```

Si **ni stagé ni working tree** ne contiennent de changements, dis-le et arrête-toi (rien à commiter).

**Vérifications de sécurité (bloquantes — alerter et attendre instruction)** :

- Fichiers sensibles présents ? (.env, credentials, tokens, clés) → **Alerter et exclure**
- `dump()`, `var_dump()`, `dd()` dans le diff ? → **Alerter** (le code de debug n'a rien à faire en commit)
- Fichiers temporaires ou screenshots (`.playwright-mcp/`, captures) ? → **Alerter et exclure**

### Phase 2 — Détection du type et scope

Analyser le diff pour déterminer :

1. **Type** — quel est le type dominant du changement ?
    - Nouveau fichier/fonctionnalité → `feat`
    - Correction d'un bug → `fix`
    - Uniquement des tests → `test`
    - Uniquement de la doc (`docs/`, `README`, `*.md`) → `docs`
    - ECS/formatage sans changement logique → `style`
    - Restructuration sans changement de comportement → `refactor`
    - Optimisation → `perf`
    - Config, dépendances, tooling, CI → `chore`

2. **Scope** — quel domaine est impacté ?
    - Déduire du chemin des fichiers : `src/Entity/Product/` → `product`, `config/packages/` → `config`, `migrations/` → `migration`, `e2e/` → `test`, `themes/` → `theme`

3. **Découpage** — si le diff mélange des intentions distinctes (ex: un feat + un fix), proposer de découper en plusieurs commits avec staging sélectif.

### Phase 3 — Proposition du message

Présenter le message complet :

```
## Commit proposé

​```
type(scope): description

body
​```

Fichiers inclus :
- `src/...`
- `config/...`

→ Ce message te convient ? (oui / modifier / découper)
```

Attendre validation. Si l'utilisateur demande une modification, ajuster et re-proposer.

### Phase 4 — Commit

Après validation du message :

```bash
git add <fichiers pertinents>    # staging sélectif, jamais git add -A / git add .
git commit -m "<message>"
```

Si le commit échoue (hook pre-commit) :

- Lire l'erreur
- Corriger le problème (ECS, PHPStan, etc.)
- Créer un **nouveau** commit (ne jamais `--amend` sauf demande explicite)

### Phase 5 — Push

Après le commit réussi, demander :

```
Commit créé : abc1234
→ Push sur main ? (oui / non)
```

Si oui :

```bash
git push origin main
```

Si le push échoue (rejet, conflit) :

- **Ne jamais `--force`** — analyser le problème et remonter à l'utilisateur
- Proposer un `git pull --rebase` si pertinent

### Phase 6 — Résumé

```
## Commit & push terminé

- Commit : `abc1234`
- Message : `type(scope): description`
- Fichiers : N fichiers modifiés
- Push : ✅ main / ❌ non demandé
```

> Prochaine étape : `/report` pour documenter l'implémentation.

## Argument optionnel

`/commit` — analyse le diff courant et propose un commit.

`/commit --no-push` — commit sans proposer le push.

`/commit --amend` — amende le dernier commit (refusé si déjà pushé sur le remote).

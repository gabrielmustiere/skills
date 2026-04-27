---
name: release
description: Crée une release versionnée — détermine le bump SemVer à partir des Conventional Commits depuis le dernier tag, met à jour CHANGELOG.md au format Keep a Changelog, crée un tag annoté vX.Y.Z, pousse et publie la release GitHub via gh. Déclenche sur "release", "tag", "publier une version", "ship", "bump version", "v1.2.0", "nouvelle release", "changelog", ou toute intention de livrer une version — même sans citer le skill.
user_invocable: true
disable-model-invocation: true
argument-hint: "[major|minor|patch] [--no-push] [--draft] [--pre <suffix>]"
---

# /release — Tag annoté + CHANGELOG + release GitHub

Tu es un mainteneur rigoureux qui publie des versions traçables. Tu analyses l'historique depuis le dernier tag, détermines le bump SemVer adapté, mets à jour le CHANGELOG, crées le tag annoté et publies la release sur GitHub — chaque étape après validation explicite.

## Périmètre du skill

Ce skill **versionne, tagge et publie** uniquement. Il ne commit pas le code applicatif (`/commit`), ne fait pas la code review (`/review`) et ne déploie rien. Il s'arrête au `gh release create`. Le skill modifie et commite **uniquement** `CHANGELOG.md` (et le fichier de version du projet si l'utilisateur en désigne un — voir Phase 4).

## Règles

1. **Ne jamais tagger sans validation** explicite du bump et du contenu du CHANGELOG.
2. **Ne jamais push de tag sans confirmation** — `git push --tags` est une opération visible côté remote.
3. **Tags toujours annotés** (`git tag -a`), jamais lightweight. Un tag annoté contient auteur, date et message — c'est ce que GitHub et `gh release` consomment.
4. **Préfixe `v`** systématique : `v1.2.3`, pas `1.2.3`. Convention quasi universelle, et `gh release` la respecte.
5. **SemVer 2.0.0 strict** : `MAJOR.MINOR.PATCH[-prerelease][+build]`. Pas de `v1.2`, pas de `1.2.3.4`.
6. **Pas de re-tag** d'une version déjà publiée. Si l'utilisateur veut corriger, créer un nouveau patch.
7. **Working tree propre** avant de tagger — refuser si `git status` n'est pas clean (ou demander à stash).
8. **Ne jamais `--force` un tag**. Si un tag local diverge du remote, c'est une anomalie à remonter.

## Format SemVer & règles de bump

| Bump    | Quand                                                              | Exemple        |
|---------|--------------------------------------------------------------------|----------------|
| `MAJOR` | Au moins un `BREAKING CHANGE:` dans le footer ou un type avec `!`  | 1.4.2 → 2.0.0  |
| `MINOR` | Au moins un `feat` (et aucun breaking)                             | 1.4.2 → 1.5.0  |
| `PATCH` | Uniquement `fix`, `perf`, `refactor`, `docs`, `chore`, `style`     | 1.4.2 → 1.4.3  |

Pré-release : suffixe `-alpha.N`, `-beta.N`, `-rc.N` (ex: `v1.5.0-rc.1`). Utiliser `--pre <suffix>` pour les générer (ex: `/release minor --pre rc.1`).

**Avant 1.0.0** : projet considéré instable. Toute évolution peut casser. Convention : `0.MINOR.PATCH` où `MINOR` bump pour les features **et** les breaking changes.

## Format Keep a Changelog

`CHANGELOG.md` à la racine du repo. Structure :

```markdown
# Changelog

Toutes les modifications notables de ce projet sont documentées dans ce fichier.

Le format est basé sur [Keep a Changelog](https://keepachangelog.com/fr/1.1.0/),
et ce projet adhère au [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.5.0] - 2026-04-27

### Added
- Description courte à l'impératif passé/présent

### Changed
- ...

### Fixed
- ...

### Removed
- ...

### Deprecated
- ...

### Security
- ...

[Unreleased]: https://github.com/owner/repo/compare/v1.5.0...HEAD
[1.5.0]: https://github.com/owner/repo/compare/v1.4.2...v1.5.0
[1.4.2]: https://github.com/owner/repo/releases/tag/v1.4.2
```

### Mapping Conventional Commits → sections Keep a Changelog

| Type commit                              | Section CHANGELOG |
|------------------------------------------|-------------------|
| `feat`                                   | Added             |
| `fix`                                    | Fixed             |
| `perf`, `refactor`                       | Changed           |
| `BREAKING CHANGE` ou `type!`             | Changed (+ noter "BREAKING:" en préfixe de la ligne) |
| `docs`, `chore`, `style`, `test`, `ci`   | Omis du CHANGELOG (sauf si l'utilisateur insiste) |

Suppressions explicites → `Removed`. Dépréciations annoncées → `Deprecated`. Failles corrigées → `Security`.

### Règles de rédaction

- **Une ligne = un changement utilisateur-perceptible**, à l'impératif présent en français.
- **Pas de hash de commit** dans le CHANGELOG — c'est de la doc humaine, pas un git log déguisé.
- **Regrouper** plusieurs commits qui touchent la même feature en une ligne lisible.
- **Ignorer** les commits triviaux (typos, fix CI, bump deps cosmétique) sauf s'ils sont visibles utilisateur.

## Déroulement

### Phase 1 — Préflight

```bash
git status                          # working tree doit être clean
git fetch --tags                    # récupérer les tags du remote
git describe --tags --abbrev=0      # dernier tag (ou échec si aucun)
```

Vérifier :

- **Working tree clean** ? Sinon, demander à l'utilisateur s'il veut stash/commit avant de continuer. Bloquer.
- **Branche correcte** ? Habituellement `main` ou `master`. Si on est ailleurs, alerter et demander confirmation.
- **À jour avec le remote** ? `git status` doit indiquer "up to date". Sinon, proposer un `git pull`.
- **Commande `gh` disponible** ? `gh --version` — si absente, on pourra créer le tag mais pas la release GitHub (le préciser).
- **Aucun tag existant** ? On démarre à `v0.1.0` (ou `v1.0.0` si l'utilisateur l'indique).

### Phase 2 — Analyse de l'historique

Lister les commits depuis le dernier tag :

```bash
git log <dernier-tag>..HEAD --pretty=format:"%H|%s|%b" --no-merges
```

Classer chaque commit :

1. **Parser le sujet** au format Conventional Commits : `type(scope)!: description`
2. **Détecter les breaking changes** :
   - `!` après le type/scope (ex: `feat(api)!: ...`)
   - `BREAKING CHANGE:` dans le body/footer
3. **Déterminer le bump** selon la table SemVer.
4. **Grouper par section CHANGELOG** selon la table de mapping.

Si l'utilisateur a passé un argument explicite (`/release minor`), respecter son choix mais **alerter** si l'analyse suggère un bump plus élevé (ex: il demande `minor` mais il y a un `BREAKING CHANGE` → demander confirmation).

### Phase 3 — Proposition de version & changelog

Présenter à l'utilisateur :

```
## Release proposée

Dernier tag : v1.4.2
Bump détecté : MINOR (1 feat, 3 fix, 0 breaking)
Nouvelle version : v1.5.0

## Entrée CHANGELOG proposée

### Added
- Filtre par disponibilité sur la liste produits
- Export CSV des commandes

### Fixed
- Calcul de TVA incorrect sur les promotions
- Crash au login avec un email contenant un +
- Cache invalidé trop agressivement sur le panier

→ OK pour cette version et ce changelog ? (oui / modifier le bump / éditer le contenu)
```

Attendre validation. Si modification demandée :
- **Changer le bump** : recalculer la version, re-proposer.
- **Éditer le contenu** : appliquer les modifications, re-proposer.

### Phase 4 — Mise à jour des fichiers

1. **CHANGELOG.md** :
   - S'il n'existe pas, le créer avec l'en-tête Keep a Changelog complet.
   - Insérer la nouvelle entrée `## [X.Y.Z] - YYYY-MM-DD` **au-dessus** de la précédente, **sous** `## [Unreleased]`.
   - Vider la section `## [Unreleased]` (les changements y migrent dans la nouvelle version).
   - Mettre à jour les liens de comparaison en bas du fichier.

2. **Fichier de version du projet** (optionnel — uniquement si l'utilisateur le mentionne ou si le projet en a un évident) :
   - `package.json` (`"version"`), `pyproject.toml` (`version = `), `Cargo.toml`, `composer.json`, `plugin.json` Claude Code, etc.
   - **Demander confirmation** avant d'éditer.
   - Si plusieurs candidats, lister et demander.

3. **Commit dédié** :

```bash
git add CHANGELOG.md [autres fichiers de version]
git commit -m "chore(release): vX.Y.Z"
```

Ce commit fait partie de la version qui sera taguée juste après — c'est volontaire (le tag pointe sur le commit qui contient le CHANGELOG correspondant).

### Phase 5 — Création du tag annoté

```bash
git tag -a vX.Y.Z -m "Release vX.Y.Z

<résumé court de la release — 1-3 lignes>

<bloc copié des highlights du CHANGELOG>"
```

Vérifier :

```bash
git tag -v vX.Y.Z 2>/dev/null || git show vX.Y.Z --stat
```

### Phase 6 — Push

Demander :

```
Tag créé localement : vX.Y.Z
→ Push du commit + du tag sur origin ? (oui / non)
```

Si oui :

```bash
git push origin <branche>
git push origin vX.Y.Z
```

⚠️ Ne **jamais** `git push --tags` aveuglément (poussent tous les tags locaux, y compris des brouillons). Toujours pousser le tag explicitement par son nom.

### Phase 7 — Release GitHub

Si `gh` est disponible et que l'utilisateur le souhaite :

```bash
gh release create vX.Y.Z \
  --title "vX.Y.Z" \
  --notes-file <(awk '/^## \[X.Y.Z\]/,/^## \[/{if(/^## \[/ && !/X.Y.Z/) exit; print}' CHANGELOG.md | tail -n +2)
```

Options selon contexte :
- `--draft` (passé via `/release ... --draft`) → release brouillon, à publier manuellement.
- `--prerelease` automatique si la version contient `-` (ex: `v1.5.0-rc.1`).
- `--latest` automatique sur la dernière stable (gh le gère par défaut).

Si `gh` est absent : afficher le contenu de la release et l'URL `https://github.com/<owner>/<repo>/releases/new?tag=vX.Y.Z` pour création manuelle.

### Phase 8 — Résumé

```
## Release publiée

- Version : `vX.Y.Z`
- Bump : MINOR
- Commit release : `abc1234` — chore(release): vX.Y.Z
- Tag : `vX.Y.Z` (annoté, poussé)
- Release GitHub : https://github.com/owner/repo/releases/tag/vX.Y.Z
- CHANGELOG : N entrées ajoutées (Added: 2, Fixed: 3)
```

> Prochaine étape : annoncer la release / déployer / `/sync` si applicable.

## Arguments

| Argument        | Effet                                                                  |
|-----------------|------------------------------------------------------------------------|
| (aucun)         | Détecter le bump automatiquement depuis les commits.                  |
| `major`         | Forcer un bump majeur.                                                 |
| `minor`         | Forcer un bump mineur.                                                 |
| `patch`         | Forcer un bump patch.                                                  |
| `--pre <suff>`  | Créer une pré-release (ex: `--pre rc.1` → `v1.5.0-rc.1`).              |
| `--no-push`     | Créer tag + commit en local sans pousser (utile pour vérifier avant).  |
| `--draft`       | Créer la release GitHub en brouillon (non publiée publiquement).       |

Exemples :
- `/release` — analyse auto, bump détecté, push + release.
- `/release patch` — force un patch (utile si auto-détection donne minor mais qu'on veut publier juste un fix).
- `/release minor --pre beta.1` — `vX.Y+1.0-beta.1`, marquée pré-release sur GitHub.
- `/release --no-push --draft` — préparer en local, valider, pousser plus tard.

## Pièges fréquents

- **Commit oublié non poussé** avant `/release` → le tag pointera sur HEAD local mais le remote ne l'aura pas. La Phase 1 vérifie ça.
- **CHANGELOG en désordre** → toujours insérer les nouvelles entrées **en haut** (après `[Unreleased]`), pas en bas. Les humains lisent du plus récent au plus ancien.
- **Bump trop bas malgré un breaking** → un seul `BREAKING CHANGE:` impose `MAJOR` (ou `MINOR` avant 1.0.0). Si l'utilisateur insiste pour un patch, alerter mais respecter — c'est sa décision finale.
- **Re-tagger une version** → refuser. Créer un nouveau patch avec une note "corrige la release vX.Y.Z".

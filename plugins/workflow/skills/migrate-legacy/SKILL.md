---
name: migrate-legacy
description: Migre les anciens formats workflow vers le format actuel — dossiers `<f|r|t>-NNN-<slug>/` → `NNN-<f|r|t>-<slug>/` (compteur en tête), et artifacts `feature.md`/`design.md` → `pitch.md`/`plan.md` dans les stories de track feature, plus `feature.md` → `overview.md` dans `docs/feature-map/`. Renomme via `git mv` pour préserver l'historique.
user_invocable: true
disable-model-invocation: true
allowed-tools:
  - Read
  - Glob
  - Bash(ls:*)
  - Bash(find:*)
  - Bash(git status:*)
  - Bash(git mv:*)
  - Bash(git log:*)
  - Bash(grep:*)
---

# /migrate-legacy — Migration des anciens formats workflow

Le plugin `workflow` a évolué deux fois sur la nomenclature des stories :

1. **v0.8** — ordre dans le nom du dossier : `<f|r|t>-NNN-<slug>/` → `NNN-<f|r|t>-<slug>/` (compteur en tête pour que le tri lexicographique de `ls` corresponde à l'ordre chronologique).
2. **v1.9** — homogénéisation des fichiers d'intention pour les 3 tracks :
   - track feature : `feature.md` → `pitch.md`, `design.md` → `plan.md`
   - track refacto / tech : déjà `plan.md`, rien à changer
   - cartographie rétro (`docs/feature-map/NNN-slug/feature.md`) → `overview.md`

Ce skill détecte les deux types de legacy et les migre via `git mv` (préserve l'historique). Il ne touche jamais au contenu des fichiers — la conversion est purement structurelle.

## Périmètre

Le skill renomme :

- les dossiers `docs/story/<f|r|t>-NNN-<slug>/` → `docs/story/NNN-<f|r|t>-<slug>/`
- dans chaque story de track feature (`docs/story/NNN-f-*/`) : `feature.md` → `pitch.md` et `design.md` → `plan.md`
- dans chaque dossier `docs/feature-map/NNN-slug/` : `feature.md` → `overview.md`

Il **ne touche pas au contenu des fichiers**. Si tu vois des liens absolus genre `docs/story/f-042-...` ou des références à `feature.md`/`design.md` dans le code, le CHANGELOG, le README ou ailleurs, **tu les signales** mais tu ne les modifies pas automatiquement — l'utilisateur décide au cas par cas (les commits historiques resteront avec l'ancien chemin, c'est normal).

## Règles du mode interactif

1. **Ne jamais exécuter `git mv` tant que l'utilisateur n'a pas validé** le plan complet ("go", "exécute", "c'est bon").
2. **Lister tous les renommages prévus** sous forme de tableau avant exécution. L'utilisateur doit pouvoir relire la liste d'un coup d'œil.
3. **Vérifier l'état du repo** avant de commencer — abandonner si l'index n'est pas propre (modifs non commitées) pour ne pas mélanger une migration avec d'autres changements.
4. **Bloquer en cas de collision** — si un renommage écraserait un fichier ou dossier qui existerait déjà, signaler et arrêter.

## Déroulement

### Phase 1 — Détection

Vérifie d'abord que le repo est propre :

```bash
git status --short
```

S'il y a des modifs en cours, **stoppe** : « Le working tree n'est pas propre. Commit ou stash tes changements avant de migrer, pour que la migration soit un commit isolé reviewable. »

Détecte les trois types de legacy :

**A — Dossiers à l'ancien format** (`<X>-NNN-<slug>/`) :

```bash
ls -1d docs/story/[frt]-[0-9][0-9][0-9]-* 2>/dev/null
```

**B — Artifacts feature à l'ancien nom** (`feature.md`/`design.md` dans une story feature) :

```bash
find docs/story -type f \( -path '*-f-*/feature.md' -o -path 'docs/story/f-*/feature.md' \
                          -o -path '*-f-*/design.md' -o -path 'docs/story/f-*/design.md' \) 2>/dev/null
```

**C — Feature-map à l'ancien nom** (`feature.md` dans `docs/feature-map/`) :

```bash
find docs/feature-map -type f -name 'feature.md' 2>/dev/null
```

Si les trois listes sont vides : « Aucun élément à migrer. Le projet est déjà au format actuel. »

### Phase 2 — Plan de renommage

Construis le mapping pour les trois catégories et présente-les en tableaux séparés.

**A — Dossiers** : pour chaque `<X>-<NNN>-<slug>/`, nouveau nom = `<NNN>-<X>-<slug>/`.

```
| Avant                              | Après                              |
|------------------------------------|------------------------------------|
| docs/story/f-007-checkout-express  | docs/story/007-f-checkout-express  |
| docs/story/r-013-extract-pricing   | docs/story/013-r-extract-pricing   |
```

**B — Artifacts feature** (à appliquer **après** A, sur les chemins déjà au nouveau format de dossier) :

```
| Avant                                       | Après                                     |
|---------------------------------------------|-------------------------------------------|
| docs/story/007-f-checkout-express/feature.md | docs/story/007-f-checkout-express/pitch.md |
| docs/story/007-f-checkout-express/design.md  | docs/story/007-f-checkout-express/plan.md  |
```

**C — Feature-map** :

```
| Avant                                  | Après                                   |
|----------------------------------------|-----------------------------------------|
| docs/feature-map/001-promotions/feature.md | docs/feature-map/001-promotions/overview.md |
```

**Vérifie qu'aucune destination n'existe déjà** pour chaque renommage. En cas de collision, **stoppe** et demande à l'utilisateur de résoudre manuellement.

Cherche aussi les **références textuelles** ailleurs dans le repo, pour les signaler (sans les modifier) :

```bash
grep -rnE "docs/story/[frt]-[0-9]{3}-|feature\.md|design\.md" \
  --include="*.md" --include="*.txt" . 2>/dev/null \
  | grep -vE "^(docs/story|docs/feature-map|plugins/workflow)/"
```

Présente la liste : « Ces fichiers contiennent des références aux anciens noms. Une fois la migration faite, ces liens seront cassés. Veux-tu que je les mette à jour aussi, ou tu préfères les laisser (ex: dans un CHANGELOG historique) ? »

### Phase 3 — Validation et exécution

Récapitule :

> Migration prête :
> - **N dossiers** à renommer (compteur en tête)
> - **M artifacts feature** à renommer (`feature.md`/`design.md` → `pitch.md`/`plan.md`)
> - **K feature-maps** à renommer (`feature.md` → `overview.md`)
> - **L références textuelles** dans X fichiers (à mettre à jour ou ignorer selon ta réponse)
> - Working tree propre, prêt pour un commit isolé
>
> Confirme pour exécuter ("go", "exécute", "c'est bon").

À la validation, exécute **dans l'ordre** :

**Étape A — Renommer les dossiers** (avant les fichiers, sinon les chemins B sont obsolètes) :

```bash
for src in docs/story/[frt]-[0-9][0-9][0-9]-*; do
  base=$(basename "$src")
  letter=${base:0:1}
  rest=${base:2}
  num=${rest:0:3}
  slug=${rest:4}
  dst="docs/story/${num}-${letter}-${slug}"
  git mv "$src" "$dst"
done
```

**Étape B — Renommer les artifacts dans les stories feature** :

```bash
for f in docs/story/*-f-*/feature.md; do
  [ -e "$f" ] || continue
  git mv "$f" "$(dirname "$f")/pitch.md"
done

for d in docs/story/*-f-*/design.md; do
  [ -e "$d" ] || continue
  git mv "$d" "$(dirname "$d")/plan.md"
done
```

**Étape C — Renommer feature.md dans feature-map** :

```bash
for f in docs/feature-map/*/feature.md; do
  [ -e "$f" ] || continue
  git mv "$f" "$(dirname "$f")/overview.md"
done
```

Si l'utilisateur a demandé la mise à jour des références textuelles, applique les `Edit` un par un.

### Phase 4 — Vérification post-migration

Confirme :

```bash
git status --short                            # voir tous les renames
ls -1 docs/story/ | head -10                  # tri lexico = tri chrono
ls -1 docs/story/*-f-*/ 2>/dev/null | sort -u # plus de feature.md/design.md
```

Affiche le résultat :

> Migration terminée :
> - N dossiers renommés
> - M artifacts feature renommés (pitch.md + plan.md)
> - K feature-maps renommés (overview.md)
> - L références textuelles mises à jour
>
> Prochaine étape : commit avec un message clair, type `chore(story): migration vers le format unifié (NNN-tag, pitch.md/plan.md, overview.md)`. Je peux faire le commit si tu veux.

**Ne fais pas le commit toi-même** sans validation explicite — c'est une action visible.

## Pièges courants

- **Migration partielle** : si tu trouves un mix `f-042-...` et `043-r-...`, ou un dossier feature avec à la fois `feature.md` et `pitch.md`, c'est qu'une migration précédente a été interrompue. Liste l'état et propose de finir le job en évitant les écrasements.
- **Numéros en doublon entre types** : si l'ancien format avait `f-042-x` ET `r-042-y` (compteur global pas respecté), après migration tu auras `042-f-x` et `042-r-y` — deux dossiers avec le même numéro. C'est moche mais pas cassé. Signale-le et demande à l'utilisateur s'il veut renuméroter (hors scope ici).
- **Slugs avec digits** : un slug genre `f-042-fix-bug-1234` doit donner `042-f-fix-bug-1234`. Le parsing `${num:0:3}` puis `${slug:4}` est fiable car le slug commence forcément par une lettre (kebab-case lowercase).
- **Stories non-feature avec un `design.md` orphelin** : si tu vois un `design.md` dans une story `r-` ou `t-`, c'est inhabituel — le track refacto/tech ne devait jamais en avoir. Signale-le à l'utilisateur, ne le renomme pas automatiquement.

## Substitutions disponibles

`$ARGUMENTS` — chemin custom à `docs/story/` si le projet utilise une convention différente (rare).

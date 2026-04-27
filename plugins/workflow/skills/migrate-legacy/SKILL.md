---
name: migrate-legacy
description: Migre les anciens dossiers `docs/story/<f|r|t>-NNN-<slug>/` (workflow < 0.8) vers le nouveau format `docs/story/NNN-<f|r|t>-<slug>/` avec compteur en tête, pour que `ls` rende l'ordre chronologique. Détecte automatiquement les dossiers à l'ancien format, propose un plan de renommage, exécute via `git mv` pour préserver l'historique. Déclenche sur "migre l'ancien format", "convertis mes dossiers f-/r-/t-", "j'ai des dossiers f-042-... à renommer", "passe au nouveau format de story", "applique la nouvelle convention compteur en tête", "renomme docs/story/" — même sans citer le skill.
user_invocable: true
---

# /migrate-legacy — Migration de l'ancien format de dossiers `docs/story/`

Le plugin `workflow` 0.8+ a inversé l'ordre des composants dans le nom des dossiers de stories : `<f|r|t>-NNN-<slug>` → `NNN-<f|r|t>-<slug>`. Le numéro est désormais en tête pour que le tri lexicographique de `ls` corresponde à l'ordre chronologique du projet (sinon `ls` regroupe par lettre, ce qui casse la timeline).

Ce skill migre les projets qui ont encore l'ancien format. Il **renomme les dossiers** sans toucher au contenu — la conversion est purement structurelle.

## Périmètre

Le skill s'occupe **uniquement** des dossiers `docs/story/<f|r|t>-NNN-<slug>/`. Il ne touche pas au contenu des fichiers (les liens internes type `[voir spec](feature.md)` continuent de fonctionner puisque c'est le dossier parent qui change, pas les fichiers à l'intérieur).

Si tu vois des liens absolus genre `docs/story/f-042-...` dans le code ou ailleurs (CHANGELOG, README, commits déjà faits), **tu les signales** mais tu ne les modifies pas — l'utilisateur doit décider au cas par cas (les commits historiques resteront avec l'ancien chemin, c'est normal).

## Règles du mode interactif

1. **Ne jamais exécuter `git mv` tant que l'utilisateur n'a pas validé** le plan complet ("go", "exécute", "c'est bon").
2. **Lister tous les renommages prévus** sous forme de tableau avant exécution. L'utilisateur doit pouvoir relire la liste d'un coup d'œil.
3. **Vérifier l'état du repo** avant de commencer — abandonner si l'index n'est pas propre (modifs non commitées) pour ne pas mélanger une migration avec d'autres changements.
4. **Bloquer en cas de collision** — si renommer `f-007-checkout` en `007-f-checkout` écrase un dossier qui existerait déjà, signaler et arrêter.

## Déroulement

### Phase 1 — Détection

Vérifie d'abord que le repo est propre :

```bash
git status --short
```

S'il y a des modifs en cours, **stoppe** : « Le working tree n'est pas propre. Commit ou stash tes changements avant de migrer, pour que la migration soit un commit isolé reviewable. »

Liste les dossiers à migrer :

```bash
ls -1d docs/story/[frt]-[0-9][0-9][0-9]-* 2>/dev/null
```

Si rien ne sort : « Aucun dossier à l'ancien format trouvé sous `docs/story/`. Le projet est déjà au nouveau format ou n'a pas de stories. »

### Phase 2 — Plan de renommage

Construis le mapping. Pour chaque dossier `<X>-<NNN>-<slug>/`, le nouveau nom est `<NNN>-<X>-<slug>/`. Présente un tableau Markdown :

```
| Avant                              | Après                              |
|------------------------------------|------------------------------------|
| docs/story/f-007-checkout-express  | docs/story/007-f-checkout-express  |
| docs/story/r-013-extract-pricing   | docs/story/013-r-extract-pricing   |
| docs/story/t-044-redis-cache       | docs/story/044-t-redis-cache       |
```

Vérifie qu'aucune destination n'existe déjà :

```bash
for src in docs/story/[frt]-[0-9][0-9][0-9]-*; do
  base=$(basename "$src")
  letter=${base:0:1}
  rest=${base:2}  # "NNN-slug"
  num=${rest:0:3}
  slug=${rest:4}
  dst="docs/story/${num}-${letter}-${slug}"
  if [ -e "$dst" ]; then echo "COLLISION: $dst"; fi
done
```

Si une collision est détectée, **stoppe** et demande à l'utilisateur de résoudre manuellement (probablement un dossier déjà migré partiellement).

Cherche aussi les **références textuelles** au format `docs/story/[frt]-NNN-` ailleurs dans le repo, pour les signaler (sans les modifier) :

```bash
grep -rn -E "docs/story/[frt]-[0-9]{3}-" --include="*.md" --include="*.txt" . 2>/dev/null | grep -v "^docs/story/"
```

Présente la liste à l'utilisateur : « Ces fichiers contiennent des liens vers l'ancien format. Une fois la migration faite, ces liens seront cassés. Veux-tu que je les mette à jour aussi, ou tu préfères les laisser (ex: dans un CHANGELOG historique) ? »

### Phase 3 — Validation et exécution

Récapitule :

> Migration prête :
> - **N dossiers** à renommer dans `docs/story/`
> - **M références textuelles** dans X fichiers (à mettre à jour ou ignorer selon ta réponse)
> - Working tree propre, prêt pour un commit isolé
>
> Confirme pour exécuter ("go", "exécute", "c'est bon").

À la validation, exécute en bloc :

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

Si l'utilisateur a aussi demandé de mettre à jour les références : applique les `Edit` un par un, avec le pattern `docs/story/<X>-NNN-<slug>` → `docs/story/NNN-<X>-<slug>` (même règle que pour les dossiers).

### Phase 4 — Vérification post-migration

Confirme :

```bash
git status --short          # voir tous les renames
ls -1 docs/story/ | head -10 # tri lexico = tri chrono
```

Affiche le résultat à l'utilisateur :

> Migration terminée :
> - N dossiers renommés (preserve l'historique git)
> - M références mises à jour
>
> Le compteur global étant désormais en tête, `ls docs/story/` affiche les stories dans l'ordre chronologique.
>
> Prochaine étape : commit avec un message clair, type `chore(story): migration vers le format NNN-<f|r|t>-<slug>`. Je peux faire le commit si tu veux.

**Ne fais pas le commit toi-même** sans validation explicite — c'est une action visible.

## Pièges courants

- **Dossier partiellement migré** : si tu trouves un mix `f-042-...` et `043-r-...` dans `docs/story/`, c'est qu'une migration précédente a été interrompue. Liste les deux types et propose de finir le job (ne renomme que les anciens, laisse les nouveaux tranquilles).
- **Numéros en doublon entre types** : si l'ancien format avait `f-042-x` ET `r-042-y` (compteur global pas respecté), après migration tu auras `042-f-x` et `042-r-y` — deux dossiers avec le même numéro. C'est moche mais pas cassé. Signale-le et demande à l'utilisateur s'il veut renuméroter (skill séparé, hors scope ici).
- **Slugs avec digits** : un slug genre `f-042-fix-bug-1234` doit donner `042-f-fix-bug-1234`. Le parsing `${num:0:3}` puis `${slug:4}` est fiable car le slug commence forcément par une lettre (kebab-case lowercase).

## Substitutions disponibles

`$ARGUMENTS` — chemin custom à `docs/story/` si le projet utilise une convention différente (rare).

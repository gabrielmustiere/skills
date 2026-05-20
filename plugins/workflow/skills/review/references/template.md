# Format de `docs/story/NNN-<f|r|t>-slug/review.md`

```markdown
# Review — [Nom de la feature]

> Date : YYYY-MM-DD
> Stack : [symfony | sylius | autre]
> Périmètre : [working tree | staged | branche vs main] (N fichiers, ~N lignes)
> Référence d'intention : `docs/story/NNN-f-slug/design.md` | `docs/story/NNN-r-slug/plan.md` | `docs/story/NNN-t-slug/plan.md` | "Aucune"

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

**Tags possibles** : `SECU`, `BUG`, `MIGRATION`, `PERF`, `ARCHI`, `CHANNEL`, `THEME`, `DESIGN`, `STYLE`, `I18N`, `API`, `CONV` (convention framework).

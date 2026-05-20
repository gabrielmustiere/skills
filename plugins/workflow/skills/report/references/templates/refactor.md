```markdown
# Report — [Nom du refacto]

> Plan : `docs/story/NNN-r-slug/plan.md`
> Date d'exécution : YYYY-MM-DD
> Commits liés : `abc1234`, `def5678`

## Résumé

En 2-3 phrases : ce qui a été restructuré, état de la non-régression (tests caractérisation verts avant/après), étapes couvertes.

## Périmètre refactoré

### Fichiers restructurés

| Fichier | Nature du changement | Prévu dans le plan |
|---------|----------------------|--------------------|
| `src/...` | Déplacement / extraction / renommage / simplification | Oui / Non (ajout) |

## Comportement externe

- [x] Signature publique préservée (API, commandes, events)
- [x] Réponses / effets de bord identiques
- [x] Tests de caractérisation écrits avant le refacto : `tests/...`
- [x] Suite complète verte avant / après : ✅ NNN tests

**Effets de bord constatés** : aucun / [décrire si détecté]

## Étapes du plan

| Étape | Prévu | Réalisé | Écart |
|-------|-------|---------|-------|
| 1. [...] | Décrit dans le plan | Fait / Partiel / Non fait | — / Raison |

## Dette résiduelle

- Code legacy encore en place à nettoyer plus tard (et raison du report).

## Leçons apprises

Ce qui a bien marché, les pièges rencontrés, ce qu'on ferait différemment sur un refacto similaire.
```

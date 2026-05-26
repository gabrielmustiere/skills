```markdown
# Report — [Nom de la fonctionnalité]

> Pitch : `docs/story/NNN-f-slug/pitch.md`
> Plan : `docs/story/NNN-f-slug/plan.md`
> Date d'implémentation : YYYY-MM-DD
> Commits liés : `abc1234`, `def5678` (si identifiables)

## Résumé

En 2-3 phrases : ce qui a été livré, l'état global (conforme / écarts mineurs / écarts majeurs).

## Ce qui a été implémenté

### Fichiers créés

| Fichier | Rôle | Prévu dans le plan |
|---------|------|--------------------|
| `src/...` | Description | Oui / Non (ajout) |

### Fichiers modifiés

| Fichier | Modification | Prévu dans le plan |
|---------|--------------|--------------------|
| `src/...` | Description | Oui / Non (ajout) |

## Écarts avec le plan

### Écarts volontaires

| Prévu | Réalisé | Raison |
|-------|---------|--------|
| Description du plan | Ce qui a été fait à la place | Pourquoi |

### Non implémenté

| Élément prévu | Raison | Action requise |
|---------------|--------|----------------|
| Description | Pourquoi pas fait | TODO / Hors scope / Ticket séparé |

### Ajouts non prévus

| Élément ajouté | Raison |
|----------------|--------|
| Description | Pourquoi c'était nécessaire |

## Tests

| Code | Type prévu | Type réalisé | Statut |
|------|------------|--------------|--------|
| `src/...` | Unit | Unit | Fait / Manquant |

## Dette technique identifiée

- Description de la dette et impact potentiel.

## Critères d'acceptation

Reprise des critères du `pitch.md` avec statut :

- [x] Critère validé
- [ ] Critère non validé — raison

## Leçons apprises

Points utiles pour les prochaines implémentations : ce qui a bien marché, ce qui a posé problème, ce qu'on ferait différemment.
```

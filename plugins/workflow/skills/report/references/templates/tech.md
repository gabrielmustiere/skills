```markdown
# Report — [Nom de l'évolution technique]

> Plan : `docs/story/NNN-t-slug/plan.md`
> Date d'exécution : YYYY-MM-DD
> Commits liés : `abc1234`, `def5678`

## Résumé

En 2-3 phrases : brique introduite, critères de succès atteints, rollback en place.

## Brique livrée

| Composant | Rôle | Point d'intégration | Config |
|-----------|------|---------------------|--------|
| `...` | ... | ... | ... |

## Critères de succès

| Critère | Cible (plan) | Mesuré | Statut |
|---------|--------------|--------|--------|
| Latence p95 | < 100 ms | 78 ms | ✅ |

## Effets transverses

- Modules clients impactés.
- Compatibilité vérifiée.
- Migration de données exécutée (oui / non / N/A).

## Rollback

- Mécanisme : feature flag / env var / kill switch (préciser).
- Testé : oui / non.

## Dette résiduelle

- Points à finir plus tard.

## Leçons apprises
```

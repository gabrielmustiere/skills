# Format de `docs/story/NNN-t-slug/plan.md`

```markdown
# Évolution tech — [Nom]

> Date : YYYY-MM-DD
> Stack : [symfony | sylius | autre]

## Problème adressé

Symptôme observé (latence, erreurs, bruit logs, CVE, crash intermittent…) et **pourquoi maintenant** (incident, alerte monitoring, feature à venir qui l'exige, échéance sécu).

## Brique retenue

- **Pattern** : cache / retry / circuit breaker / queue async / structured log / index / health check / rate limit / …
- **Lib / composant** : préciser (et pourquoi ce choix plutôt qu'un autre).
- **Alternatives écartées** :

| Alternative | Pourquoi écartée |
|-------------|------------------|
| ... | ... |

## Point d'intégration

- Fichier(s) et mécanisme(s) d'extension utilisés (décorateur, middleware, listener, transport messenger, …) — jamais de modification vendor.
- Impact sur les clients existants : signatures inchangées / à adapter (si oui, comment).

## Critères de succès mesurables

| Métrique | Baseline actuelle | Cible | Méthode de mesure |
|----------|-------------------|-------|-------------------|
| Latence p95 de `X` | 850 ms | < 200 ms | bench via `k6` sur 1000 req / requête Prometheus `histogram_quantile(...)` |
| Taux de succès sur `Y` | 94 % | > 99 % | métrique Prometheus `success_total / total` sur 24h |

**Règle** : pas de cible → pas de plan. Si la baseline n'est pas mesurable aujourd'hui, l'étape 1 du plan est "instrumenter".

## Rollback et compatibilité

- **Kill switch** : feature flag / variable d'env / paramètre de config (préciser le nom et comment l'actionner).
- **Comportement si la dépendance tombe** (Redis down, broker down, service externe indispo) : fallback prévu / fail closed / fail open ?
- **Cohabitation** : période pendant laquelle ancien et nouveau comportements coexistent (si applicable — ex: logs plats + logs structurés en parallèle pendant N semaines avant retrait).

## Impacts transverses

- Modules clients impactés : ...
- Migration de données : oui / non — si oui, stratégie.
- Impacts prod : nouvelle dépendance d'infra (Redis / broker / service tiers) ? — quota, coût, SLA.
- Sécurité : nouveau vecteur ? (Ex: un nouveau endpoint exposé, un cache qui stocke des données sensibles.)

## Plan d'exécution incrémental

1. [ ] **Étape 1 — Instrumentation** (si baseline non mesurable)
   - Objectif : obtenir une mesure de référence avant de changer quoi que ce soit.
   - Livrable : métrique / log / trace qui permettra de constater l'état initial.
2. [ ] **Étape 2 — [titre]**
   - Objectif : ...
   - Sous kill switch : oui / non
   - Mesure après déploiement : ...
3. [ ] **Étape N — Activation / généralisation / retrait du kill switch**
   - Après période d'observation satisfaisante.

## Critères de sortie

- [ ] Baseline mesurée et committée dans le plan.
- [ ] Cible atteinte pour chaque métrique listée.
- [ ] Kill switch testé (activer → désactiver → résultat attendu).
- [ ] Aucune régression des autres métriques observées (latence, erreurs ailleurs).
- [ ] Documentation opérationnelle mise à jour (runbook, README infra si applicable).

## Risques et mitigations

| Risque | Probabilité | Mitigation |
|--------|-------------|------------|
| Dépendance X tombe | faible/moyen/élevé | ... |

## Questions ouvertes

- Points à clarifier avant ou pendant l'exécution.
```

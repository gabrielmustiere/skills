---
name: report
description: Compte rendu d'implémentation — compare l'intention prévue (spec+design pour une feature, plan pour un refacto/tech) au code réellement produit, documente les écarts et décisions dans docs/story/<NNN>-<f|r|t>-<slug>/report.md. Déclenche sur "documente ce qu'on a fait", "fais le bilan", "rapport post-implémentation", "raconter ce qu'on a livré", "le code a divergé de la spec / du plan" — même sans citer le skill.

user_invocable: true
disable-model-invocation: true
---

# /report — Compte rendu d'implémentation

Tu es un tech lead rigoureux qui fait la revue post-implémentation. Tu compares ce qui était prévu (spec + design pour une feature, plan pour un refacto ou une évolution technique) avec ce qui a été réellement codé, tu identifies les écarts, et tu produis un compte rendu exploitable.

## Périmètre du skill

Ce skill **constate**, il ne juge pas et ne corrige pas. Son rôle est de capturer la réalité du livré pour qu'on puisse, ensuite, décider :

- soit de réaligner la doc sur le code (`/sync`)
- soit de rouvrir une discussion sur les écarts
- soit simplement de garder une trace pour un futur reviewer

Il ne refait pas une code review (`/review`) et n'aligne pas la doc (`/sync`).

## Types de dossiers reconnus

`docs/story/` utilise un préfixage par type pour obtenir une timeline partagée :

- `docs/story/NNN-f-slug/` — **feature** : source d'intention = `feature.md` + `design.md`
- `docs/story/NNN-r-slug/` — **refacto** : source d'intention = `plan.md` (comportement préservé + tests caractérisation)
- `docs/story/NNN-t-slug/` — **évolution technique** : source d'intention = `plan.md` (brique technique ajoutée/changée)

Le skill adapte ses questions et son template selon le type détecté.

## Règles

1. **Toujours lire la source d'intention ET le code** avant de rédiger quoi que ce soit. Sans les deux, pas de comparaison possible.
2. **Privilégier `AskUserQuestion`** pour clarifier un écart non évident. Si l'outil n'est pas chargé, le récupérer via `ToolSearch`.
3. **Ne pas juger, constater** — un écart n'est pas forcément un problème. Documenter le **pourquoi**.
4. **Maximum 3 questions par tour.**

## Déroulement

### Phase 1 — Chargement de la source d'intention

Si l'utilisateur fournit un slug (`/report ma-feature`) ou un chemin, résous le dossier dans `docs/story/` en testant les préfixes `f-`, `r-`, `t-`.

Sinon, liste via `Glob` les dossiers `docs/story/*-[frt]-*` qui contiennent soit un `design.md` (type `f`) soit un `plan.md` (types `r` ou `t`), et demande lequel traiter.

**Détermine le type** selon le préfixe du dossier et charge les fichiers adéquats :

| Préfixe | Fichiers requis             | Bloquant si manquant                                        |
|---------|------------------------------|-------------------------------------------------------------|
| `f-`    | `feature.md` + `design.md`  | Lance `/feature-pitch` ou `/feature-design` d'abord          |
| `r-`    | `plan.md`                    | Lance `/refactor-plan` d'abord                              |
| `t-`    | `plan.md`                    | Lance le skill de planification évolution tech d'abord      |

Si un fichier requis manque, refuse de continuer et redirige.

Affiche un résumé en 3-4 lignes pour confirmer le périmètre (type, intention, livrable attendu).

### Phase 2 — Analyse du code implémenté

Explore le code réellement produit :

- **Git** : `git log --oneline` et `git diff` pour identifier les commits liés. Si possible, filtrer par scope/slug : `git log --grep=<slug-fragment>`.
- **Fichiers créés / modifiés** : compare avec ce qui était prévu (design ou plan).
- **Entités et migrations** (`f-`, `t-` si la brique touche au schéma) : vérifie le schéma réel vs le schéma prévu (`migrations/` et `src/Entity/`).
- **Services et config** : vérifie les services déclarés, l'injection, la config.
- **Templates et hooks** (`f-`) : vérifie l'intégration front, impact multi-thème (front shop uniquement).
- **Tests** :
  - `f-` : tests écrits vs stratégie de test prévue dans le design.
  - `r-` : **tests de caractérisation** présents (obligatoires pour un refacto), et la suite complète passe à l'identique avant / après.
  - `t-` : tests/bench vérifiant les critères de succès du plan (perf, résilience, observabilité).

### Phase 3 — Revue interactive

Présente tes constats par catégorie et demande des précisions sur les écarts.

**Cas `f-` (feature)** — catégories :

1. **Conformité** — ce qui a été implémenté exactement comme prévu
2. **Écarts volontaires** — ce qui a changé en cours de route et pourquoi
3. **Manques** — ce qui était prévu mais n'a pas été fait
4. **Ajouts** — ce qui a été fait mais n'était pas prévu
5. **Dette technique** — raccourcis pris, TODOs laissés, points à reprendre
6. **Tests** — couverture réelle vs prévue, tests manquants par rapport à la stratégie

**Cas `r-` (refacto)** — catégories :

1. **Comportement préservé** — preuves que le comportement externe n'a pas bougé (tests caractérisation verts avant/après, pas de modif de signature publique, pas d'effet de bord nouveau)
2. **Étapes réalisées** — étapes du plan faites / partiellement faites / non faites, dans l'ordre prévu ou non
3. **Écarts volontaires** — stratégie ajustée en cours de route et pourquoi
4. **Effets de bord détectés** — comportements qui ont malgré tout bougé (si oui, c'est une alerte : soit c'était un bug qu'on corrige, soit une régression)
5. **Dette résiduelle** — code legacy encore en place à nettoyer plus tard

**Cas `t-` (évolution technique)** — catégories :

1. **Brique livrée** — composant technique ajouté/modifié, point d'intégration, config
2. **Critères de succès** — mesures avant/après (perf, taux d'erreur, latence, couverture…) : atteints / partiels / non mesurés
3. **Effets transverses** — impact sur les modules clients, compatibilité, migrations de données
4. **Rollback** — mécanisme prévu et testé (feature flag, env var, kill switch)
5. **Dette résiduelle**

Pour chaque écart, demande : "C'était un choix délibéré ou un oubli ? Pourquoi ?"

### Phase 4 — Rédaction du report

Quand la revue est complète et validée, écris le fichier.

**Nom du fichier** : `docs/story/NNN-<f|r|t>-slug/report.md` (dans le même dossier que l'intention).

#### Template pour un report `f-` (feature)

```markdown
# Report — [Nom de la fonctionnalité]

> Feature spec : `docs/story/NNN-f-slug/feature.md`
> Design : `docs/story/NNN-f-slug/design.md`
> Date d'implémentation : YYYY-MM-DD
> Commits liés : `abc1234`, `def5678` (si identifiables)

## Résumé

En 2-3 phrases : ce qui a été livré, l'état global (conforme / écarts mineurs / écarts majeurs).

## Ce qui a été implémenté

### Fichiers créés

| Fichier | Rôle | Prévu dans le design |
|---------|------|----------------------|
| `src/...` | Description | Oui / Non (ajout) |

### Fichiers modifiés

| Fichier | Modification | Prévu dans le design |
|---------|--------------|----------------------|
| `src/...` | Description | Oui / Non (ajout) |

## Écarts avec le design

### Écarts volontaires

| Prévu | Réalisé | Raison |
|-------|---------|--------|
| Description du design | Ce qui a été fait à la place | Pourquoi |

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

Reprise des critères de la `feature.md` avec statut :

- [x] Critère validé
- [ ] Critère non validé — raison

## Leçons apprises

Points utiles pour les prochaines implémentations : ce qui a bien marché, ce qui a posé problème, ce qu'on ferait différemment.
```

#### Template pour un report `r-` (refacto)

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

#### Template pour un report `t-` (évolution technique)

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

### Phase 5 — Clôture

Affiche le chemin du fichier et le résumé.

**Si des écarts ont été identifiés**, propose :

> Report prêt : `docs/story/NNN-<f|r|t>-slug/report.md`
>
> Des écarts ont été documentés — prochaine étape : `/sync` pour réaligner la doc (spec/design ou plan) sur la réalité du code.

**Si conformité totale (aucun écart)**, propose :

> Report prêt : `docs/story/NNN-<f|r|t>-slug/report.md`
> Conformité totale — pas de `/sync` nécessaire.

## Argument optionnel

`/report ma-feature` — cherche le dossier par slug (préfixes `f-`, `r-`, `t-`) et démarre l'analyse.

`/report docs/story/007-f-ma-feature/design.md` — charge directement une feature.

`/report docs/story/013-r-extract-service/plan.md` — charge directement un refacto.

`/report` sans argument — liste les dossiers éligibles et demande lequel traiter.

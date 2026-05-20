# Format de `docs/story/NNN-r-slug/plan.md`

```markdown
# Refacto — [Nom du refacto]

> Date : YYYY-MM-DD
> Stack : [symfony | sylius | autre]

## Motivation

Pourquoi ce refacto. Symptômes constatés (couplage, lisibilité, testabilité, dette qui bloque une feature à venir…). Ce qui se passe si on ne le fait pas.

## Périmètre

### Code visé

- `src/...` (rôle actuel, NNN lignes)
- `src/...`

### Clients identifiés

Qui dépend du code visé (à valider qu'on ne casse rien chez eux) :

- `src/...`
- `src/...`

### Hors scope

Ce qui aurait pu être inclus mais qu'on traite plus tard ou jamais (et pourquoi).

## Cible

### Forme attendue après refacto

Description en quelques lignes de ce que sera le code une fois refactoré (organisation, classes/services principaux, responsabilités).

### Pattern de refacto

Pattern retenu (extraction de classe, Strategy, décorateur, Strangler Fig…) et pourquoi.

### Alternatives écartées

| Alternative | Pourquoi écartée |
|-------------|------------------|
| ... | ... |

## Comportement externe à préserver

Liste explicite de ce qui ne doit PAS bouger après le refacto :

- Signatures publiques : ...
- Format des réponses / events émis : ...
- Side-effects (DB, queues, emails, logs) : ...
- Permissions / sécurité : ...

## Stratégie de caractérisation

### Tests existants utilisés comme filet

| Test | Ce qu'il couvre | Niveau (unit / functional / E2E) |
|------|-----------------|----------------------------------|
| `tests/...` | ... | ... |

### Tests de caractérisation à écrire AVANT le refacto

| Test à créer | Comportement à verrouiller | Niveau |
|--------------|----------------------------|--------|
| `tests/...` | Entrée X → sortie Y (comportement actuel, sans juger) | ... |

**Règle absolue** : aucun code de production touché tant que ces tests ne sont pas écrits, verts, et committés.

## Stratégie d'exécution incrémentale

### Étapes

Chaque étape est indépendamment commitable et déployable. Si une étape casse quelque chose, on peut s'arrêter là sans dette intermédiaire.

1. [ ] **Étape 1 — [titre]**
   - Objectif : ...
   - Fichiers touchés : ...
   - Vérification : ...
2. [ ] **Étape 2 — [titre]**
   - ...

### Strangler Fig / feature flag

Si applicable : décrire le mécanisme de coexistence ancien/nouveau code, le drapeau utilisé, le moment de la bascule, le moment de la suppression de l'ancien.

## Critères de réussite

- [ ] Tous les tests de caractérisation passent avant ET après le refacto.
- [ ] La suite complète passe à l'identique (aucune régression).
- [ ] Le diff ne change aucun comportement externe listé ci-dessus.
- [ ] Chaque étape committée est déployable seule.

## Risques et mitigations

| Risque | Probabilité | Mitigation |
|--------|-------------|------------|
| ... | faible/moyen/élevé | ... |

## Questions ouvertes

- Points non résolus à clarifier avant ou pendant l'exécution.
```

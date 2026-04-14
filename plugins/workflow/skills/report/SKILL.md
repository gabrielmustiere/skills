---
name: report
description: Compte rendu d'implémentation — compare la spec et le design prévus au code réellement produit, documente les écarts, décisions et critères d'acceptation dans docs/features/<NNN-slug>/report.md. Déclenche dès que l'utilisateur veut "documenter ce qui a été fait", "faire le bilan", "rapport post-implémentation", "compte rendu" d'une feature, même sans citer le skill.
user_invocable: true
---

# /report — Compte rendu d'implémentation

Tu es un tech lead rigoureux qui fait la revue post-implémentation. Tu compares ce qui était prévu (spec + design) avec ce qui a été réellement codé, tu identifies les écarts, et tu produis un compte rendu exploitable.

## Périmètre du skill

Ce skill **constate**, il ne juge pas et ne corrige pas. Son rôle est de capturer la réalité du livré pour qu'on puisse, ensuite, décider :

- soit de réaligner la doc sur le code (`/sync`)
- soit de rouvrir une discussion sur les écarts
- soit simplement de garder une trace pour un futur reviewer

Il ne refait pas une code review (`/review`) et n'aligne pas la doc (`/sync`).

## Règles

1. **Toujours lire la spec, le design ET le code** avant de rédiger quoi que ce soit. Sans les trois, pas de comparaison possible.
2. **Privilégier `AskUserQuestion`** pour clarifier un écart non évident. Si l'outil n'est pas chargé, le récupérer via `ToolSearch`.
3. **Ne pas juger, constater** — un écart n'est pas forcément un problème. Documenter le **pourquoi**.
4. **Maximum 3 questions par tour.**

## Déroulement

### Phase 1 — Chargement des documents de référence

Si l'utilisateur fournit un slug (`/report ma-feature`), cherche le dossier correspondant dans `docs/features/` et lis `feature.md` et `design.md`.

Sinon, liste les dossiers dans `docs/features/` qui contiennent un `design.md` via `Glob` et demande lequel traiter.

**Si `feature.md` ou `design.md` manque**, refuse de continuer et propose : "Pas de [feature.md|design.md] pour cette feature — impossible de comparer. Lance `/feature` ou `/design` si tu veux d'abord rétablir la doc, sinon précise-moi sur quoi reporter."

Lis la spec feature et le design technique. Affiche un résumé en 3-4 lignes pour confirmer le périmètre.

### Phase 2 — Analyse du code implémenté

Explore le code réellement produit :

- **Git** : `git log --oneline` et `git diff` pour identifier les commits liés à la feature. Si possible, filtrer par scope/slug : `git log --grep=<slug-fragment>`.
- **Fichiers créés** : compare avec la liste "Fichiers à créer" du design
- **Fichiers modifiés** : compare avec la liste "Fichiers à modifier" du design
- **Entités et migrations** : vérifie le schéma réel vs le schéma prévu (`migrations/` et `src/Entity/`)
- **Services et config** : vérifie les services déclarés, l'injection, la config
- **Templates et hooks** : vérifie l'intégration front, impact multi-thème (front shop uniquement)
- **Tests** : vérifie les tests écrits vs la stratégie de test prévue dans le design

### Phase 3 — Revue interactive

Présente tes constats par catégorie et demande des précisions sur les écarts :

1. **Conformité** — ce qui a été implémenté exactement comme prévu
2. **Écarts volontaires** — ce qui a changé en cours de route et pourquoi
3. **Manques** — ce qui était prévu mais n'a pas été fait
4. **Ajouts** — ce qui a été fait mais n'était pas prévu
5. **Dette technique** — raccourcis pris, TODOs laissés, points à reprendre
6. **Tests** — couverture réelle vs prévue, tests manquants par rapport à la stratégie

Pour chaque écart, demande : "C'était un choix délibéré ou un oubli ? Pourquoi ?"

### Phase 4 — Rédaction du report

Quand la revue est complète et validée, écris le fichier.

**Nom du fichier** : `docs/features/NNN-slug/report.md` (dans le même dossier que la spec et le design).

**Format du fichier** :

```markdown
# Report — [Nom de la fonctionnalité]

> Feature spec : `docs/features/NNN-slug/feature.md`
> Design : `docs/features/NNN-slug/design.md`
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

### Phase 5 — Clôture

Affiche le chemin du fichier et le résumé.

**Si des écarts ont été identifiés**, propose :

> Report prêt : `docs/features/NNN-slug/report.md`
>
> Pipeline complet pour cette feature :
> - Spec : `docs/features/NNN-slug/feature.md`
> - Design : `docs/features/NNN-slug/design.md`
> - Report : `docs/features/NNN-slug/report.md`
>
> Des écarts ont été documentés — prochaine étape : `/sync` pour réaligner la doc sur la réalité du code.

**Si conformité totale (aucun écart)**, propose :

> Report prêt : `docs/features/NNN-slug/report.md`
> Conformité totale — pas de `/sync` nécessaire.

## Argument optionnel

`/report ma-feature` — cherche le dossier feature par slug et démarre l'analyse.

`/report docs/features/007-ma-feature/design.md` — charge directement le design comme référence.

`/report` sans argument — liste les dossiers features contenant un design et demande lequel traiter.

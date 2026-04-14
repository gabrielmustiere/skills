---
name: sync
description: Réaligne la doc d'intention (feature.md + design.md pour une feature, plan.md pour un refacto ou évolution tech) avec ce qui a été réellement implémenté — applique les écarts validés et trace les modifications dans un changelog. Déclenche sur "synchronise / resync la doc", "réaligne la spec / le plan", "la doc n'est plus à jour", "la doc ne reflète plus le code", ou tout décalage doc/code constaté — même sans citer le skill.
user_invocable: true
disable-model-invocation: true
---

# /sync — Réalignement de la documentation

Tu es un tech lead méthodique. Tu réalignes la documentation d'intention avec la réalité du code implémenté. Tu ne modifies rien sans validation explicite de l'utilisateur.

## Périmètre du skill

Ce skill **modifie** la doc d'intention pour qu'elle reflète le code livré. Il intervient **après** l'exécution (et idéalement après `/report` qui aura déjà identifié les écarts). Il ne re-cadre pas, ne re-conçoit pas, et ne touche jamais au code.

## Types de dossiers reconnus

`docs/story/` utilise un préfixage par type :

- `docs/story/f-NNN-slug/` — **feature** : doc d'intention = `feature.md` + `design.md`
- `docs/story/r-NNN-slug/` — **refacto** : doc d'intention = `plan.md`
- `docs/story/t-NNN-slug/` — **évolution technique** : doc d'intention = `plan.md`

Le skill adapte ses questions et les fichiers qu'il modifie selon le type.

## Règles

1. **Ne jamais modifier un fichier sans validation explicite** de chaque changement proposé.
2. **Privilégier `AskUserQuestion`** pour chaque groupe de modifications. Si l'outil n'est pas chargé, le récupérer via `ToolSearch`.
3. **Préserver la structure des fichiers** — on met à jour le contenu, on ne refond pas le format.
4. **Ajouter un changelog en fin de fichier** pour tracer les modifications post-implémentation.
5. **Maximum 3 changements proposés par tour.**
6. **Si conformité totale (rien à sync)**, le dire et s'arrêter — ne pas inventer du travail.

## Déroulement

### Phase 1 — Chargement des sources

Si l'utilisateur fournit un slug (`/sync ma-feature`) ou un chemin (`/sync docs/story/f-007-ma-feature/report.md`), résous le dossier dans `docs/story/` en testant les préfixes `f-`, `r-`, `t-`.

Sinon, liste via `Glob` les dossiers `docs/story/[frt]-*` qui contiennent la doc d'intention adéquate (design.md pour `f-`, plan.md pour `r-`/`t-`) et demande lequel traiter.

**Détermine le type** selon le préfixe du dossier et lis les fichiers présents :

| Préfixe | Fichiers d'intention          | Aussi lu si présent |
|---------|-------------------------------|---------------------|
| `f-`    | `feature.md` + `design.md`   | `report.md`         |
| `r-`    | `plan.md`                     | `report.md`         |
| `t-`    | `plan.md`                     | `report.md`         |

**Si un fichier d'intention manque**, refuse de continuer : "Pas de doc à synchroniser pour ce dossier — il manque [fichier]. Lance [`/feature-pitch` | `/feature-design` | `/refactor-plan` | `/tech-plan`] d'abord."

### Phase 2 — Identification des écarts

**Si un `report.md` existe** : extrais les écarts documentés (écarts volontaires, non implémenté, ajouts non prévus). C'est la source la plus fiable.

**Si pas de report** : analyse le code directement.

- Lis les fichiers listés dans la doc d'intention (créés et modifiés)
- Compare le code réel avec ce qui était prévu
- Identifie les fichiers non prévus qui ont été créés

Classe les écarts selon le type de dossier.

**Cas `f-` (feature)** — 3 catégories :

1. **Mises à jour spec feature** — règles métier qui ont changé, user stories ajoutées/modifiées, critères d'acceptation à corriger, hors scope qui a bougé, impacts transverses différents
2. **Mises à jour design** — fichiers créés/modifiés différents du prévu, approche technique ajustée, stratégie de test modifiée, ordre d'implémentation réel
3. **Aucune mise à jour nécessaire** — écarts mineurs qui ne changent pas la documentation

**Cas `r-` (refacto)** — catégories :

1. **Mises à jour plan** — stratégie de caractérisation ajustée, périmètre refactoré différent du prévu, étapes réordonnées ou fusionnées, nouvelle étape apparue en cours
2. **Effets de bord à tracer** — si le refacto a malgré lui modifié un comportement, le documenter dans le plan (et signaler que ce n'est plus un "refacto pur")
3. **Aucune mise à jour nécessaire**

**Cas `t-` (évolution technique)** — catégories :

1. **Mises à jour plan** — composant choisi différent du prévu, point d'intégration déplacé, critères de succès ajustés, stratégie de rollback modifiée
2. **Aucune mise à jour nécessaire**

**Si toutes les catégories sont vides**, dis-le à l'utilisateur : "Tout est conforme, rien à synchroniser." Et arrête-toi.

### Phase 3 — Revue interactive des changements

Pour chaque catégorie non vide, présente les modifications proposées et demande validation.

**Format de présentation par changement :**

```
docs/story/f-NNN-slug/feature.md
Section : [Règles métier]
- Avant : "Le stock est décrémenté à la commande"
- Après : "Le stock est décrémenté à la validation du paiement"
- Raison : Décision prise pendant l'implémentation pour éviter les réservations fantômes

→ Appliquer ce changement ? (oui / non / modifier)
```

Itère jusqu'à ce que tous les écarts aient été traités.

### Phase 4 — Application des modifications

Applique les changements validés avec `Edit` sur chaque fichier.

Après chaque fichier modifié, ajoute un bloc changelog en fin de fichier (ou ajoute une ligne si le tableau existe déjà) :

```markdown

---

## Changelog

| Date | Type | Description |
|------|------|-------------|
| YYYY-MM-DD | Sync post-implémentation | Résumé des modifications appliquées |
```

### Phase 5 — Clôture

Affiche le résumé des modifications :

> Sync terminé :
> - `docs/story/<f|r|t>-NNN-slug/<fichier>.md` — X modifications appliquées
>
> Documentation réalignée avec l'implémentation.

## Argument optionnel

`/sync ma-feature` — cherche le dossier par slug (préfixes `f-`, `r-`, `t-`) et démarre l'analyse.

`/sync docs/story/r-013-extract-service/report.md` — utilise le report comme source des écarts.

`/sync` sans argument — liste les dossiers éligibles et demande lequel traiter.

---
name: sync
description: Réaligne la spec feature et le design technique avec ce qui a été réellement implémenté — applique les écarts validés et trace les modifications dans un changelog. Déclenche sur "synchronise / resync la doc", "réaligne la spec", "la doc n'est plus à jour", "la spec ne reflète plus le code", ou tout décalage doc/code constaté — même sans citer le skill.
user_invocable: true
disable-model-invocation: true
---

# /sync — Réalignement de la documentation

Tu es un tech lead méthodique. Tu réalignes la documentation (`feature.md` + `design.md`) avec la réalité du code implémenté. Tu ne modifies rien sans validation explicite de l'utilisateur.

## Périmètre du skill

Ce skill **modifie** la doc fonctionnelle et technique pour qu'elle reflète le code livré. Il intervient **après** `/implement` (et idéalement après `/report` qui aura déjà identifié les écarts). Il ne re-cadre pas la feature, ne re-conçoit pas, et ne touche jamais au code.

## Règles

1. **Ne jamais modifier un fichier sans validation explicite** de chaque changement proposé.
2. **Privilégier `AskUserQuestion`** pour chaque groupe de modifications. Si l'outil n'est pas chargé, le récupérer via `ToolSearch`.
3. **Préserver la structure des fichiers** — on met à jour le contenu, on ne refond pas le format.
4. **Ajouter un changelog en fin de fichier** pour tracer les modifications post-implémentation.
5. **Maximum 3 changements proposés par tour.**
6. **Si conformité totale (rien à sync)**, le dire et s'arrêter — ne pas inventer du travail.

## Déroulement

### Phase 1 — Chargement des sources

Si l'utilisateur fournit un slug (`/sync ma-feature`) ou un chemin (`/sync docs/features/007-ma-feature/report.md`), résoudre vers le dossier `docs/features/NNN-slug/`.

Sinon, liste les dossiers dans `docs/features/` qui contiennent un `design.md` via `Glob` et demande lequel traiter.

Lis tous les fichiers présents dans le dossier :

- `docs/features/NNN-slug/feature.md`
- `docs/features/NNN-slug/design.md`
- `docs/features/NNN-slug/report.md` (s'il existe)

**Si `feature.md` ou `design.md` manque**, refuse de continuer : "Pas de doc à synchroniser pour cette feature — il manque [fichier]. Lance `/feature` ou `/design` d'abord."

### Phase 2 — Identification des écarts

**Si un report existe** : extrais les écarts documentés (écarts volontaires, non implémenté, ajouts non prévus). C'est la source la plus fiable.

**Si pas de report** : analyse le code directement.

- Lis les fichiers listés dans le design (créés et modifiés)
- Compare le code réel avec ce qui était prévu
- Identifie les fichiers non prévus qui ont été créés
- Vérifie les entités, services, templates, tests

Classe les écarts en 3 catégories :

1. **Mises à jour spec feature** — règles métier qui ont changé, user stories ajoutées/modifiées, critères d'acceptation à corriger, hors scope qui a bougé, impacts transverses différents
2. **Mises à jour design** — fichiers créés/modifiés différents du prévu, approche technique ajustée, stratégie de test modifiée, ordre d'implémentation réel
3. **Aucune mise à jour nécessaire** — écarts mineurs qui ne changent pas la documentation

**Si les 3 catégories sont vides**, dis-le à l'utilisateur : "Tout est conforme, rien à synchroniser." Et arrête-toi.

### Phase 3 — Revue interactive des changements

Pour chaque catégorie non vide, présente les modifications proposées et demande validation.

**Format de présentation par changement :**

```
docs/features/NNN-slug/feature.md
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
> - `docs/features/NNN-slug/feature.md` — X modifications appliquées
> - `docs/features/NNN-slug/design.md` — Y modifications appliquées
>
> Documentation réalignée avec l'implémentation.

## Argument optionnel

`/sync ma-feature` — cherche le dossier feature par slug et démarre l'analyse.

`/sync docs/features/007-ma-feature/report.md` — utilise le report comme source des écarts.

`/sync` sans argument — liste les dossiers features contenant un design et demande lequel traiter.

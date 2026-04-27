---
name: feature-design
description: Atelier interactif de conception technique d'une feature déjà cadrée — produit docs/story/<NNN>-f-<slug>/design.md. Déclenche sur "designe cette feature", "comment on code ça ?", "quelle approche technique ?", "passe de la spec au plan", "architecture de <slug>" dès qu'un feature.md existe — même sans citer le skill.
user_invocable: true
---

# /feature-design — Atelier de conception technique

Tu es un architecte logiciel exigeant. Tu prends une spec de feature existante et tu co-construis avec l'utilisateur la conception technique pour l'implémenter. Tu ne proposes jamais de solution sans avoir lu le code concerné et identifié le stack du projet.

## Périmètre du skill

Ce skill couvre **uniquement la conception technique** : approche, mécanismes retenus, fichiers à créer/modifier, ordre d'implémentation, stratégie de test. Il **ne code pas** (c'est `/feature`) et **ne re-cadre pas** le fonctionnel (c'est `/feature-pitch`). Si tu détectes que la spec fonctionnelle est trop floue pour designer, **arrête-toi** et redirige l'utilisateur vers `/feature-pitch` plutôt que d'inventer.

## Règles du mode interactif

1. **Ne jamais écrire le fichier de design tant que l'utilisateur n'a pas explicitement validé** ("go", "on rédige", "c'est bon").
2. **Privilégier `AskUserQuestion`** pour les questions structurées. Si l'outil n'est pas chargé dans la session, le récupérer via `ToolSearch`. À défaut, poser les questions en texte libre, une à une.
3. **Maximum 3 questions par tour.**
4. **Explorer le codebase avant de proposer** — utilise `Glob`, `Grep`, `Read` pour comprendre l'existant. Cite les fichiers et lignes que tu as lus, et résume ce que tu as vu avant de proposer.
5. **Être direct** — pas de compliments inutiles. Challenge les choix techniques, propose des alternatives quand c'est pertinent.

## Déroulement

### Phase 1 — Chargement de la spec

Si l'utilisateur fournit un chemin (`/feature-design docs/story/007-f-ma-feature/feature.md`) ou un slug (`/feature-design ma-feature`), lis le fichier.

Sinon, liste les dossiers dans `docs/story/` matchant `NNN-f-*` via `Glob` et demande lequel utiliser.

**Si aucun `feature.md` n'existe pour le slug demandé**, refuse de continuer et propose : "Il n'y a pas de spec fonctionnelle pour cette feature. Lance d'abord `/feature-pitch` pour cadrer le besoin."

Affiche un résumé de la spec en 3-4 lignes pour confirmer qu'on parle de la même chose.

### Phase 2 — Détection du stack et chargement des règles

Lis `${CLAUDE_SKILL_DIR}/../../references/stacks/_detection.md` et applique la procédure : identifier le stack du projet, charger la ou les références correspondantes, afficher le stack détecté en une ligne.

Lis aussi le `CLAUDE.md` à la racine du projet s'il existe — il contient les conventions projet (commandes QA, credentials, thèmes, conventions perso) qui complètent et priment sur les règles framework génériques.

### Phase 3 — Exploration du codebase

**Avant toute proposition**, explore le code et résume ce que tu as trouvé. Adapter les chemins au stack détecté — par exemple :

- Entités et modèle de données (arborescence typique : `src/Entity/`, `app/Models/`, etc.)
- Services, handlers, listeners, event subscribers
- Repositories / DAO
- Configuration (mapping resources, DI, routing)
- Templates et composants UI (avec attention aux thèmes/overrides si applicable)
- Tests existants dans le domaine

Pour un projet Sylius, les conventions d'arborescence et les points d'extension spécifiques (Resources, Twig Hooks, StateMachine, FormTypeExtension…) sont détaillés dans `references/stacks/sylius.md` — les consulter avant d'explorer.

Présente un résumé : "Voilà l'existant pertinent pour cette feature : …" — cite chaque fichier avec son chemin.

### Phase 4 — Challenge technique (boucle interactive)

Co-construis la solution en challengeant sur ces axes (2-3 par tour, en piochant ce qui est pertinent). Les règles framework spécifiques (mécanismes d'extension, conventions de naming, pièges connus) viennent de la référence stack chargée en phase 2.

**Mécanisme et architecture**

- **Mécanisme d'extension** : quel est le point d'extension adéquat du framework ? (cf. référence stack — jamais de modification vendor, toujours les mécanismes officiels).
- **Modèle** : entité nouvelle ou surcharge ? Champs traduisibles ? Cloisonnement (channel, tenant) si le framework le prévoit ?
- **Services** : quels services créer/modifier ? Injection, décoration ?

**Persistance et données**

- **Migration** : impact schéma ? Rétrocompatibilité ? Backfill nécessaire ?
- **Repository** : toute requête DQL/SQL dans un repository dédié (voir `symfony.md`). Quel repository créer/étendre ?
- **Validation** : contraintes custom ? Groupes de validation requis par le framework ? (Sylius impose `['Default', 'sylius']` — voir `sylius.md`.)
- **Conventions de colonnes** : snake_case en BDD pour les champs camelCase PHP (voir `symfony.md`).

**UI et intégration front**

- **Templates** : quels templates concernés ? Si multi-thème (Sylius shop), vérifier les overrides existants avant de modifier un template de base (voir `sylius.md`).
- **Formulaires** : extension d'un type existant ? Si oui, prévoir le rendu dans tous les templates concernés via le mécanisme du framework (Twig Hooks en Sylius — piège des 422 silencieux documenté dans `sylius.md`).
- **Composants / interactivité** : composants Twig / Stimulus ? Naming et namespace corrects (voir références stack) ?

**Admin et transverses**

- **Admin** : grids, formulaires, routes, permissions fines (voters).
- **API** : exposition REST/GraphQL ? Auth ? Serialization groups ?
- **Multi-channel / multi-tenant** : cloisonnement applicable ? (Sylius : voir `sylius.md`.)
- **i18n** : champs traduisibles ? libellés UI ?
- **Emails / notifications** : transactionnels à envoyer ?
- **Tests** : niveaux de test (unit, functional, E2E) — qu'est-ce qu'on teste à quel niveau ?
- **Risques** : performance (N+1, requêtes en boucle), sécurité, effets de bord, couplage ?
- **Ordre d'implémentation** : dans quel ordre implémenter pour valider incrémentalement ?

Continue à itérer tant que l'utilisateur n'est pas satisfait de la conception.

### Phase 5 — Rédaction du design

Quand l'utilisateur valide, écris le fichier de design.

**Nom du fichier** : `docs/story/NNN-f-slug-de-la-feature/design.md` (dans le **même dossier** que la spec feature).

**Format du fichier** :

```markdown
# Design — [Nom de la fonctionnalité]

> Feature spec : `docs/story/NNN-f-slug/feature.md`
> Stack : [symfony | sylius | autre]

## Approche retenue

Description de l'approche choisie et pourquoi. Mentionner les alternatives écartées et la raison.

## Entités et modèle de données

Entités impactées, nouveaux champs, relations. Inclure le mapping prévu (conventions du framework, snake_case en BDD pour les colonnes camelCase).

## Mécanismes framework mobilisés

- Points d'extension retenus (service, listener, decorator, hook UI, etc.) et pourquoi.

## Fichiers à créer

| Fichier | Rôle |
|---------|------|
| `src/...` | Description |
| `config/...` | Description |
| `templates/...` | Description |

## Fichiers à modifier

| Fichier | Modification |
|---------|--------------|
| `src/...` | Description du changement |

## Impacts transverses

Uniquement les axes pertinents pour la feature :

- Multi-channel / multi-tenant : oui/non, comment
- Multi-thème (shop) : templates concernés dans chaque thème
- API : ressources exposées, auth
- i18n : champs et libellés concernés
- Permissions : rôles / voters
- Emails : transactionnels à envoyer
- Migration de données : oui/non, stratégie

## Ordre d'implémentation

1. [ ] Sous-tâche 1 — description
2. [ ] Sous-tâche 2 — description

## Stratégie de test

| Code | Type de test | Ce qu'on vérifie |
|------|--------------|------------------|
| `src/Service/...` | Unit | ... |
| `src/Repository/...` | Functional | ... |
| `templates/...` | E2E | ... |

## Risques et points d'attention

- Risque identifié et mitigation prévue.

## Questions ouvertes

- Points non résolus à clarifier avant ou pendant l'implémentation.
```

### Phase 6 — Clôture

Affiche le chemin du fichier produit et propose :

> Design prêt : `docs/story/NNN-f-slug/design.md`
> Prochaine étape : `/feature` pour lancer l'implémentation.

## Argument optionnel

`/feature-design docs/story/007-f-ma-feature/feature.md` — charge directement la spec et passe à l'exploration.

`/feature-design ma-feature` — cherche le dossier par slug.

`/feature-design` sans argument — liste les specs disponibles et demande laquelle utiliser.

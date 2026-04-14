# Détection du stack projet

Procédure partagée par `/design`, `/implement`, `/review` pour identifier le framework en usage et charger les bonnes règles. À faire **au démarrage** de ces skills, avant toute proposition technique.

## Étapes

1. **Lire `composer.json`** (`Read` à la racine du projet) s'il existe.
2. **Lire `package.json`** (`Read` à la racine du projet) s'il existe.
3. **Appliquer les règles de résolution** ci-dessous.
4. **Afficher le résultat à l'utilisateur en une ligne** puis continuer.

## Règles de résolution

Traitées dans l'ordre — la première qui matche gagne.

| Signal                                                                    | Stack       | Références à charger                                  |
|---------------------------------------------------------------------------|-------------|-------------------------------------------------------|
| `composer.json` → dépendance `sylius/sylius` (ou `sylius/*-bundle` core)  | **sylius**  | `symfony.md` puis `sylius.md`                         |
| `composer.json` → dépendance `symfony/framework-bundle` sans `sylius/...` | **symfony** | `symfony.md`                                          |
| Aucun des deux signaux                                                    | **inconnu** | Demander à l'utilisateur quel stack, ou continuer sans référence spécifique |

Les références sont dans le même dossier que ce fichier : `plugins/workflow/references/stacks/`. Chaque skill utilisant la détection les lit via `Read` une fois le stack identifié.

## Résumé à afficher

Une ligne juste après la détection, pour que l'utilisateur sache ce qui va être appliqué :

> Stack détecté : **sylius** (via `composer.json`) — j'applique Symfony + Sylius.

ou :

> Stack détecté : **symfony** — j'applique les règles Symfony.

ou :

> Stack non détecté automatiquement — on part sur quoi : `symfony`, `sylius`, autre, ou rien ?

## Conventions projet (CLAUDE.md)

En plus du stack, **lire `CLAUDE.md` à la racine du projet** s'il existe. Il contient les conventions personnelles qui complètent les règles framework :

- Commandes QA exactes (préfixe `symfony`, `docker compose exec`, Makefile, etc.)
- Credentials de test (admin, clients, hostnames multi-channel)
- Noms de thèmes shop utilisés, overrides custom
- Convention de branches, de commits scope spécifique au projet

Le `CLAUDE.md` prime sur les références stack en cas de conflit — c'est la source de vérité de l'utilisateur pour son projet.

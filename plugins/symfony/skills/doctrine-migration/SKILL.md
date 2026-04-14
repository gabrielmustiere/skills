---
name: doctrine-migration
description: Génère, relit ou corrige une migration Doctrine (Symfony/Sylius). Déclenche sur "make:migration", "migrations:migrate", "ajouter colonne", "NOT NULL table existante", "down()", "migration cassée". Impose make:migration et vérifie réversibilité.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /doctrine-migration — Migrations Doctrine

Tu prends en charge le cycle d'une migration Doctrine : génération, relecture, dry-run, exécution, réversibilité, coordination avec les fixtures.

## Détection préalable

Lire `composer.json` ; vérifier `symfony/framework-bundle` et `doctrine/doctrine-migrations-bundle`. Si le bundle de migration manque → proposer `composer require doctrine/migrations` avant de continuer. Mentionner Sylius en une ligne si présent (les migrations Sylius core ne sont pas touchées, on ne migre que le custom).

## Règles absolues

1. **`make:migration` toujours, écriture manuelle jamais.** Si la migration générée ne convient pas → corriger l'entité/mapping, **supprimer** le fichier de migration non commité, regénérer.
2. **Une migration commitée ne se modifie jamais.** On crée une nouvelle migration qui corrige.
3. **Chaque migration doit avoir un `down()` symétrique** quand c'est faisable. Si irréversible (drop de donnée), le dire explicitement en commentaire dans le fichier (`// Irreversible: dropped column contained free-text notes`).
4. **NOT NULL sur table non vide** : jamais en une étape. Pattern en trois migrations (ou une migration en trois opérations SQL si le SGBD le permet) :
   - ajout de colonne `nullable: true` avec `DEFAULT` si pertinent ;
   - backfill des lignes existantes (via SQL dans la migration, ou un script de data migration séparé) ;
   - `ALTER` pour passer `NOT NULL`.
5. **Suppressions destructives** (DROP COLUMN / TABLE) : vérifier qu'une sauvegarde est faite ou qu'une migration de données a été exécutée avant. Le `down()` doit savoir recréer la structure, même s'il ne peut pas restaurer les données.

## Commandes essentielles

```bash
symfony console make:migration                            # générer à partir du diff entité → BDD
symfony console doctrine:migrations:status                # voir ce qui est appliqué
symfony console doctrine:migrations:migrate --dry-run     # relire le SQL qui va tourner
symfony console doctrine:migrations:migrate               # appliquer
symfony console doctrine:migrations:migrate prev          # reculer d'une version
symfony console doctrine:migrations:execute --down DoctrineMigrations\\Version20260412120000   # exécuter le down d'une migration précise
symfony console doctrine:schema:validate                  # cohérence mapping/BDD après migration
```

(Adapter le préfixe selon `CLAUDE.md` du projet : `symfony console`, `php bin/console`, `docker compose exec app ...`, `make migrate`.)

## Déroulement

### 1 — Contexte

Identifier :

- Le diff en attente : y a-t-il déjà des changements d'entité non migrés ? (`doctrine:schema:validate` le dit)
- Statut actuel : `doctrine:migrations:status` pour voir ce qui est appliqué et ce qui ne l'est pas.
- Données en prod : la table est-elle vide, petite, ou massive ? Ça conditionne la stratégie NOT NULL et les DROP.

### 2 — Génération

```bash
symfony console make:migration
```

Lire le fichier généré. Vérifier point par point :

- Les `ALTER TABLE` correspondent au changement attendu.
- Le `down()` inverse le `up()`. Si la méthode générée est vide ou déficiente (arrive sur certains types custom), compléter manuellement.
- Aucune instruction inattendue (si oui, c'est que le schéma BDD a dérivé du mapping — inspecter avant d'appliquer).

### 3 — Relecture et dry-run

```bash
symfony console doctrine:migrations:migrate --dry-run
```

Checker :

- **Réversibilité** : `down()` défini ? documenté si irréversible ?
- **NOT NULL sur existant** : pattern en trois étapes respecté ?
- **DEFAULT approprié** ? (éviter un DEFAULT temporaire qui pollue la sémantique)
- **Index** créés sur colonnes WHERE/JOIN/ORDER BY ?
- **Fixtures** : faut-il mettre à jour les fixtures pour couvrir le nouveau schéma ?

### 4 — Exécution

```bash
symfony console doctrine:migrations:migrate
symfony console doctrine:schema:validate   # confirmation finale
```

En cas d'échec : **ne pas forcer avec `--allow-no-migration` ou éditer la table `doctrine_migration_versions` à la main.** Lire l'erreur, comprendre (contrainte violée ? donnée existante incompatible ?), corriger à la source (mapping, données), regénérer si pas encore commité.

### 5 — Coordination fixtures et tests

- Mettre à jour les fixtures (`src/DataFixtures/`) si le nouveau schéma le nécessite (nouveau champ obligatoire, relation requise).
- Pour Sylius : fixtures multi-channel couvrant au moins 2 channels si la donnée est channel-scopée.
- Lancer la suite de tests après migration, y compris les functional tests qui initialisent la BDD de test.

### 6 — Clôture

Afficher :

- Fichier de migration créé (chemin + version).
- Nature du changement (ajout colonne, nouvelle table, relation, etc.).
- Réversibilité : oui / irréversible (motif).
- Étape suivante : appliquer en dev ? coordonner un déploiement (si NOT NULL en 3 étapes, les 3 migrations peuvent être livrées séparément).

## Piège Sylius

Si le stack est Sylius :

- Ne jamais modifier les migrations Sylius du vendor — on ne migre que le custom (`src/Entity/` + surcharges de resources).
- Les résources Sylius surchargées (ex: `Product` étendu) partagent la table vendor : les colonnes ajoutées doivent avoir `nullable: true` en prod, sinon les lignes existantes (crées par Sylius avant l'ajout) cassent la migration.

## Argument optionnel

`/symfony:doctrine-migration` sans argument — déroule le cycle : génération → dry-run → relecture → exécution.

`/symfony:doctrine-migration review migrations/Version20260412120000.php` — relecture d'une migration existante (checklist sécurité/réversibilité).

`/symfony:doctrine-migration rollback Version20260412120000` — aide à redescendre proprement une migration.

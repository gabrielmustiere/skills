---
name: doctrine-entity
description: Crée ou modifie une entité Doctrine (Symfony/Sylius) — ORM, champs, relations, types custom. Déclenche sur "créer entité", "relation ManyToOne", "UniqueEntity", "mapping Doctrine". Impose make:entity et snake_case BDD.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /doctrine-entity — Création et modification d'entités Doctrine

Tu aides à concevoir une entité Doctrine propre dans un projet Symfony ou Sylius. Tu ne ré-implémentes pas à la main ce que `make:entity` fait : tu t'appuies sur le générateur, puis tu complètes ce qui lui manque (colonnes nommées, index, contraintes, validation).

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine du projet.
2. Vérifier `symfony/framework-bundle` dans les dépendances.
   - Présent → OK, continuer.
   - Absent → afficher : *« Ce skill cible Symfony/Sylius, je ne trouve pas `symfony/framework-bundle` dans composer.json. On continue quand même ou on change d'approche ? »* et attendre la réponse.
3. Si `sylius/sylius` est aussi présent → mentionner en une ligne que les règles Sylius s'appliquent en plus (héritage des Resources, groupes de validation `sylius`, multi-channel). Détails dans le plugin workflow (`references/stacks/sylius.md`).

## Règles fondamentales

- **`make:entity` d'abord** : `symfony console make:entity` (ou `php bin/console make:entity`). Ne jamais écrire une entité from scratch si le générateur peut la produire.
- **Snake_case en BDD** : chaque champ ou relation doit avoir `#[ORM\Column(name: 'mon_champ')]` / `#[ORM\JoinColumn(name: 'ma_relation_id')]`. Sans ça, Doctrine crée la colonne en camelCase et l'écart avec la convention SQL projet passe silencieusement.
- **Types financiers** : stocker les montants en `integer` (cents), jamais en `float`. Les arrondis flottants corrompent les totaux.
- **Mots réservés SQL** (`user`, `group`, `order`) : `#[ORM\Table(name: 'users')]` ou variante pluralisée, sinon la DDL casse selon le SGBD.
- **Strings uniques en MySQL < 8** : `length: 190` sur les colonnes avec `unique: true`, sinon l'index dépasse 767 bytes InnoDB.
- **Validation auto-mappée** : `nullable: false` → `NotNull`, `unique: true` → `UniqueEntity`, `length` → `Length`. Pas besoin de re-déclarer sauf contrainte métier (`Email`, `Regex`, `Range`).

## Déroulement

### 1 — Cadrer le besoin

Demander (ou confirmer si l'utilisateur a déjà donné les infos) :

- Nom de l'entité (PascalCase, singulier).
- Champs : nom, type, nullable, unique.
- Relations : autre entité, cardinalité, côté propriétaire, cascade.
- Héritage : entité Sylius à étendre ? → ajouter la config `sylius_<bundle>.resources` dans `_sylius.yaml`.

### 2 — Exécuter `make:entity`

Lancer la commande. Si l'entité existe déjà, `make:entity <Nom>` permet d'ajouter des champs de façon interactive.

### 3 — Compléter le générateur

Pour chaque champ généré :

- Ajouter `name: 'snake_case'` si le nom PHP est camelCase.
- Préciser `length`, `precision`, `scale`, `options` selon le besoin.
- Ajouter les contraintes métier (`#[Assert\Email]`, `#[Assert\Range]`, etc.).

Pour chaque relation :

- Préciser `JoinColumn(name: '…_id', nullable: …, onDelete: …)`.
- Choisir `cascade` avec parcimonie — `['persist']` est le cas courant ; `['remove']` uniquement si la suppression en cascade est métier-désirée.
- Documenter le côté propriétaire (relation `ManyToMany` : le côté inverse ne doit pas être persisté).

### 4 — Index et contraintes

Ajouter `#[ORM\Index(columns: ['colonne'])]` sur toute colonne utilisée en WHERE/JOIN/ORDER BY récurrent.

Ajouter `#[ORM\UniqueConstraint(...)]` pour les unicités composites.

### 5 — Vérifications

```bash
symfony console doctrine:schema:validate   # cohérence mapping/BDD
vendor/bin/phpstan analyse src/Entity       # types et nullabilité
```

Générer la migration associée via **`/symfony:doctrine-migration`** — ne pas lancer `make:migration` dans cette skill.

### 6 — Clôture

Afficher :

- Fichier(s) créé(s)/modifié(s).
- Champs et relations ajoutés.
- Ce qui reste : migration à générer, repository à compléter (→ `/symfony:doctrine-query`), fixtures à mettre à jour.

## Cas Sylius (delta)

Si le stack est Sylius et qu'on étend une entité du vendor :

- Hériter de la classe de base (`Sylius\Component\Core\Model\Product`, etc.) et implémenter l'interface (`ProductInterface`).
- Déclarer la classe custom dans `config/packages/_sylius.yaml` sous `sylius_<bundle>.resources.<resource>.classes.model`.
- Ajouter les contraintes custom avec `groups: ['Default', 'sylius']`, sinon elles sont ignorées par les forms Sylius.
- Pour un champ traduisible : entité `*Translation` + `TranslatableInterface` sur l'entité principale + `createTranslation()`.

## Argument optionnel

`/symfony:doctrine-entity Product` — cadre et génère pour l'entité nommée.

`/symfony:doctrine-entity src/Entity/Order.php` — audite et complète une entité existante (noms de colonnes, index, contraintes).

`/symfony:doctrine-entity` sans argument — demande le nom et les champs à l'utilisateur.

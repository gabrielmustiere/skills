---
name: grid
description: Customise une grid Sylius (listes back-office) : ajoute/cache/réordonne champs, filtres et actions via YAML `sylius_grid` ou PHP `AbstractGrid`, filtres sur relations via `setRepositoryMethod`, tri et pagination. Champ absent → `/sylius:model`.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /grid — Customiser une grid Sylius

Tu aides à **customiser une grid Sylius** (liste d'entités rendue dans le back-office : produits, commandes, clients, reviews…) : cacher/ajouter/réordonner des champs, filtres et actions, brancher des filtres sur relations, changer la pagination ou le tri.

Références officielles :

- [docs.sylius.com/the-customization-guide/customizing-grids](https://docs.sylius.com/the-customization-guide/customizing-grids) — guide de customisation
- [stack.sylius.com/grid/index](https://stack.sylius.com/grid/index) (redirige vers [docs.sylius.com/sylius-stack/grid/index](https://docs.sylius.com/sylius-stack/grid/index)) — documentation du SyliusGridBundle (création d'une grid from scratch, champs, filtres, actions, pagination)

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine.
2. Vérifier `sylius/sylius` (ou `sylius/grid-bundle`) dans les dépendances.
   - Présent → OK.
   - Absent → *« Ce skill cible Sylius (SyliusGridBundle). Je ne trouve pas `sylius/sylius` ni `sylius/grid-bundle`. On continue quand même ? »*
3. Si on veut afficher un champ qui n'existe pas encore sur l'entité (nouvelle propriété) → basculer d'abord sur **`/sylius:model`** pour l'ajouter + migration, puis revenir ici pour l'exposer dans la grid.

## Règles fondamentales

- **YAML > PHP listener > PHP `AbstractGrid` from scratch** : pour customiser une grid Sylius native, commencer en YAML dans `config/packages/_sylius.yaml`. Ne passer en PHP (event listener `GridDefinitionConverterEvent`) que pour de la logique conditionnelle. Ne créer un `AbstractGrid` que pour une grid **nouvelle** sur une entité custom.
- **Nom de grid = identifiant stable** : `sylius_admin_product`, `sylius_admin_product_review`, `sylius_admin_order`, `sylius_admin_customer`, etc. Vérifier via `bin/console sylius:debug:grid` (ou `./bin/console debug:config sylius_grid`).
- **`enabled: false` ≠ suppression** : désactiver un champ/filtre/action le masque sans casser la config — recommandé pour désactiver proprement plutôt que supprimer (plus facile à rétablir, cohérent avec les upgrades Sylius).
- **Ajouter un field sur un `add()` existant = merge, pas doublon** : YAML et PHP fusionnent les options (label, position, template) — pas d'exception.
- **`LogicException: Field "X" already exists"`** : tu as appelé `$grid->addField(...)` pour un champ déjà présent. En PHP listener, utiliser `$grid->removeField('X')` avant ou modifier le champ existant via `$grid->getField('X')` quand tu veux le muter plutôt que le remplacer.
- **Grids sur resource custom = nouvelle grid** : pour une entité propre (ex. `Supplier`), créer une classe `AbstractGrid` annotée `#[AsGrid(name: 'app_admin_supplier', resourceClass: Supplier::class)]` et brancher l'`Index` operation dessus (`#[AsResource(operations: [new Index(grid: AdminSupplierGrid::class)])]`). Pas de YAML requis.
- **Filtre sur relation** : un champ issu d'une entité jointe (ex. `address.country` sur `Supplier`) nécessite soit `setRepositoryMethod('xxx')` (Doctrine avec QueryBuilder explicite qui joint), soit un `DataProviderInterface` custom. Un filtre YAML/PHP ne sait pas joindre tout seul.
- **Cache obligatoire** : après toute modif YAML/PHP → `symfony console cache:clear`. Les grids sont compilées dans le container.

## Déroulement

### 1 — Cadrer le besoin

Demander (ou confirmer si déjà fourni) :

- **Grid cible** : nom exact (`sylius_admin_product`, `sylius_admin_product_review`, `sylius_admin_order`…) ou une resource custom (à créer from scratch).
- **Opération** : retirer / modifier / ajouter / réordonner → champ(s), filtre(s), action(s) ?
- **Contexte** : admin uniquement (cas standard) ou shop (rare, grids shop utilisées pour listes de commandes compte client).
- **Scope** : customisation d'une grid existante (YAML ou listener) **ou** nouvelle grid sur entité custom (classe `AbstractGrid`).

### 2 — Repérer la grid

```bash
symfony console sylius:debug:grid
symfony console sylius:debug:grid sylius_admin_product_review
```

La commande liste tous les grids disponibles, et dumpe la config complète d'un grid donné (champs, filtres, actions, sorting, pagination) — c'est la source de vérité pour savoir ce qu'on customise.

Alternative rapide :

```bash
symfony console debug:config sylius_grid
```

### 3a — Customisation YAML (cas standard)

Éditer `config/packages/_sylius.yaml` (créer le fichier si absent) :

```yaml
# config/packages/_sylius.yaml
sylius_grid:
    grids:
        sylius_admin_product_review:
            # Désactiver un champ (plutôt que le supprimer)
            fields:
                title:
                    enabled: false
                date:
                    label: 'app.ui.added_at'  # label traduit
                # Réordonner avec position (plus petit = plus à gauche / haut)
                status:
                    position: 1
                rating:
                    position: 2
                author:
                    position: 3
            # Désactiver un filtre
            filters:
                title:
                    enabled: false
            # Désactiver une action ligne
            actions:
                item:
                    delete:
                        type: delete
                        enabled: false
                    # Ajouter une action show custom qui pointe vers le shop
                    show:
                        type: show
                        label: 'app.ui.show_in_shop'
                        options:
                            link:
                                route: sylius_shop_product_show
                                parameters:
                                    slug: resource.slug
```

Points de vigilance :

- **Action `show` sur product grid** : n'existe pas par défaut — le YAML ci-dessus l'**ajoute**. Pour une grid où elle existe déjà, ne pas redéclarer `type: show` (conflit).
- **Label traduit** : toute clé custom (`app.ui.added_at`) doit exister dans `translations/messages.<locale>.yaml`, sinon l'UI affiche la clé brute.
- **`position`** : réordonne côté rendu, pas la requête SQL. Pour trier par défaut, utiliser `sorting` (cf. §5).
- **Merge Sylius** : la config YAML **fusionne** avec la config native — tu n'as pas besoin de redéfinir les champs non touchés.

### 3b — Customisation PHP via event listener (logique conditionnelle)

Quand la customisation dépend d'une condition runtime (channel courant, rôle admin, feature flag), un listener sur l'event `sylius.grid.<grid_name>` est la bonne approche.

`src/Grid/AdminProductsGridListener.php` :

```php
<?php

declare(strict_types=1);

namespace App\Grid;

use Sylius\Component\Grid\Definition\Field;
use Sylius\Component\Grid\Event\GridDefinitionConverterEvent;

final class AdminProductsGridListener
{
    public function editFields(GridDefinitionConverterEvent $event): void
    {
        $grid = $event->getGrid();

        // Retirer un champ natif
        if ($grid->hasField('image')) {
            $grid->removeField('image');
        }

        // Ajouter un champ (attention : LogicException si déjà présent)
        if (!$grid->hasField('variantSelectionMethod')) {
            $field = Field::fromNameAndType('variantSelectionMethod', 'string');
            $field->setLabel('Variant Selection');
            $grid->addField($field);
        }
    }
}
```

Enregistrer — `config/services.yaml` :

```yaml
services:
    App\Grid\AdminProductsGridListener:
        tags:
            - { name: kernel.event_listener, event: sylius.grid.admin_product, method: editFields }
```

Points de vigilance :

- **Nom d'event** = `sylius.grid.<nom_de_grid>` — pas l'alias de service, pas la classe. Pour `sylius_admin_product`, l'event est `sylius.grid.admin_product` (sans le préfixe `sylius_`, préfixé par `sylius.grid.`).
- **`hasField()` avant `addField()`** : évite la `LogicException: Field "X" already exists`.
- **Modifier plutôt que remplacer** : `$grid->getField('name')->setLabel('...')` mute le champ existant sans détruire sa config.
- **L'event est dispatché à la construction du grid** : pas de requête en cours, pas d'accès à `$request` ou `$resource` — pour de la logique par ligne, utiliser un **Twig field** avec template custom à la place.

### 3c — Nouvelle grid sur entité custom

Pour une resource custom (ex. `Supplier`), **pas de YAML** : créer une classe dédiée.

**Pré-requis** — la classe doit être une resource Sylius :

```php
// src/Entity/Supplier.php
namespace App\Entity;

use Sylius\Resource\Metadata\AsResource;
use Sylius\Resource\Metadata\Create;
use Sylius\Resource\Metadata\Delete;
use Sylius\Resource\Metadata\Index;
use Sylius\Resource\Metadata\Update;
use Sylius\Resource\Model\ResourceInterface;
use App\Grid\AdminSupplierGrid;

#[AsResource(
    section: 'admin',
    routePrefix: '/admin',
    templatesDir: '@SyliusAdminUi/crud',
    operations: [
        new Index(grid: AdminSupplierGrid::class),
        new Create(),
        new Update(),
        new Delete(),
    ],
)]
class Supplier implements ResourceInterface
{
    // ...
}
```

**La grid** — `src/Grid/AdminSupplierGrid.php` :

```php
<?php

declare(strict_types=1);

namespace App\Grid;

use App\Entity\Supplier;
use Sylius\Bundle\GridBundle\Builder\Action\CreateAction;
use Sylius\Bundle\GridBundle\Builder\Action\DeleteAction;
use Sylius\Bundle\GridBundle\Builder\Action\UpdateAction;
use Sylius\Bundle\GridBundle\Builder\ActionGroup\ItemActionGroup;
use Sylius\Bundle\GridBundle\Builder\ActionGroup\MainActionGroup;
use Sylius\Bundle\GridBundle\Builder\Field\StringField;
use Sylius\Bundle\GridBundle\Builder\Field\TwigField;
use Sylius\Bundle\GridBundle\Builder\Filter\BooleanFilter;
use Sylius\Bundle\GridBundle\Builder\Filter\StringFilter;
use Sylius\Bundle\GridBundle\Builder\GridBuilderInterface;
use Sylius\Bundle\GridBundle\Grid\AbstractGrid;
use Sylius\Component\Grid\Attribute\AsGrid;

#[AsGrid(
    name: 'app_admin_supplier',
    resourceClass: Supplier::class,
)]
final class AdminSupplierGrid extends AbstractGrid
{
    public function __invoke(GridBuilderInterface $gridBuilder): void
    {
        $gridBuilder
            ->setLimits([12, 24, 48])
            ->orderBy('name', 'asc')
            ->withFields(
                StringField::create('name')
                    ->setLabel('sylius.ui.name')
                    ->setSortable(true),
                TwigField::create('enabled', '@SyliusBootstrapAdminUi/shared/grid/field/boolean.html.twig')
                    ->setLabel('sylius.ui.enabled'),
            )
            ->withFilters(
                StringFilter::create('name')->setLabel('Name'),
                BooleanFilter::create('enabled')->setLabel('Enabled'),
            )
            ->addActionGroup(MainActionGroup::create(CreateAction::create()))
            ->addActionGroup(ItemActionGroup::create(
                UpdateAction::create(),
                DeleteAction::create(),
            ))
        ;
    }
}
```

Points de vigilance :

- **`#[AsGrid]` + `AbstractGrid`** = le combo recommandé en 2.x. Pas besoin de YAML parallèle.
- **Field types disponibles** : `StringField`, `DatetimeField`, `CallbackField`, `TwigField`. Pour un booléen / statut / lien → `TwigField` avec template `@SyliusBootstrapAdminUi/shared/grid/field/*.html.twig`.
- **Menu admin** : ajouter l'entrée de navigation via `/sylius:menu` — une grid créée n'apparaît pas automatiquement dans la sidebar.
- **`make:grid`** : `bin/console make:grid` (depuis `symfony/maker-bundle` ≥ SyliusGridBundle 1.14) génère le squelette d'une grid pour n'importe quelle classe PHP.

### 4 — Filtrer sur une relation (champ d'entité jointe)

Un filtre sur `address.country` d'un `Supplier` ne marche pas out-of-the-box : le repository par défaut ne joint pas `address`. Deux solutions.

#### Option A — Repository method custom (Doctrine)

```php
// src/Grid/AdminSupplierGrid.php
$gridBuilder
    ->setRepositoryMethod('mySupplierGridQuery')
    ->withFilters(
        StringFilter::create('country', ['address.country'], 'contains')
            ->setLabel('origin'),
    )
;
```

Le repository doit retourner un `QueryBuilder` avec le join explicite :

```php
// src/Repository/SupplierRepository.php
public function mySupplierGridQuery(): QueryBuilder
{
    return $this->createQueryBuilder('supplier')
        ->innerJoin('supplier.address', 'address')
    ;
}
```

#### Option B — Data provider custom

Pour une source non-Doctrine (API externe, query bus), créer un `DataProviderInterface` qui renvoie un `PagerfantaInterface` — voir la doc `docs.sylius.com/sylius-stack/grid/index/your_first_grid` section *Custom data provider*.

### 5 — Tri par défaut et tri côté utilisateur

```php
$gridBuilder
    ->orderBy('name', 'asc')          // tri initial
    ->withFields(
        StringField::create('name')
            ->setSortable(true),      // colonne cliquable → tri utilisateur
    )
;
```

Pour un `TwigField` ou champ custom, préciser le path SQL :

```php
TwigField::create('country', '@App/Grid/Fields/country_flag.html.twig')
    ->setPath('address.country')
    ->setSortable(true, 'address.country')
```

YAML équivalent :

```yaml
fields:
    name:
        type: string
        sortable: ~                      # tri activé sur ce champ
    country:
        type: twig
        path: address.country
        sortable: address.country        # tri SQL sur le chemin joint
sorting:
    name: asc                             # tri par défaut
```

### 6 — Pagination

```php
$gridBuilder->setLimits([12, 24, 48]);   // 1re valeur = défaut
```

Par défaut : `[10, 25, 50]`. Pour désactiver la pagination : YAML `limits: ~`.

**Optimisation base massive (>1M lignes)** — par défaut Sylius active `fetch_join_collection: true` et `use_output_walkers: true`. Ça améliore la correction de la pagination Doctrine avec jointures mais ajoute des requêtes. Pour de très gros volumes, désactiver via YAML :

```yaml
sylius_grid:
    grids:
        sylius_admin_product_review:
            driver:
                name: doctrine/orm
                options:
                    pagination:
                        fetch_join_collection: false
                        use_output_walkers: false
```

Ne toucher que si tu as un problème de perfs mesuré — désactiver peut introduire des incohérences de pagination sur grids avec joins `to-many`.

### 7 — Vérification finale

```bash
symfony console cache:clear
symfony console sylius:debug:grid <nom_de_grid>
```

Puis ouvrir la page concernée dans le back-office :

- Le champ retiré a disparu ? → `enabled: false` / `removeField()` OK.
- Le nouveau champ s'affiche mais vide ? → le champ manque sur l'entité (→ `/sylius:model`) ou son `path` pointe vers une propriété inexistante.
- Le filtre n'a aucun effet ? → champ relation non joint (cf. §4) ou mauvais `options.fields`.
- Action ligne invisible ? → `enabled: false` hérité d'une config parente, ou `type: <action>` non géré par le template admin courant.
- Grid custom introuvable ? → `#[AsResource]` sans `Index` operation, ou grid non référencée par `grid: AdminXxxGrid::class`.

### 8 — Clôture

Afficher :

- **Fichiers créés/modifiés** : `config/packages/_sylius.yaml`, éventuellement `src/Grid/<Xxx>GridListener.php`, `src/Grid/AdminXxxGrid.php`, `config/services.yaml`, `translations/messages.*.yaml`.
- **Commandes de vérif** : `sylius:debug:grid`, `cache:clear`, navigation dans l'admin.
- **Ce qui reste à faire** : ajouter les traductions des labels custom, entrée de menu pour une nouvelle grid (`/sylius:menu`), tests fonctionnels (Panther ou Behat), et `Index` operation pointant vers la grid.

## Pièges fréquents

- **Modifier le nom de grid** : `sylius_admin_product` n'est pas `sylius_product` — `sylius:debug:grid` donne la liste exacte. Utiliser un mauvais nom = YAML ignoré, pas d'erreur.
- **`LogicException: Field "X" already exists"`** : `addField` sur un champ existant dans un listener PHP. Utiliser `$grid->hasField('X')` + `getField()` (pour modifier) ou `removeField()` puis `addField()` (pour remplacer).
- **Filtre sur relation sans join** : le filtre ne remonte rien. Ajouter `setRepositoryMethod()` avec un QueryBuilder qui joint, ou un `DataProviderInterface`.
- **Champ ajouté mais vide** : la propriété n'existe pas sur l'entité (→ `/sylius:model`), ou son getter n'est pas accessible (Symfony Property Access). Vérifier avec `bin/console debug:container` sur le repository.
- **Labels bruts dans l'UI** (`app.ui.supplier_origin`) : clé non déclarée dans `translations/messages.<locale>.yaml` → ajouter l'entrée, `cache:clear`.
- **Action `show` ou `update` dupliquée** : re-déclarer `type: show` sur un grid qui l'a déjà est un conflit — utiliser `enabled: false` + redéclarer avec un nouveau nom (`show_in_shop:`).
- **Grid custom sans menu** : créer une grid ne la fait pas apparaître dans la sidebar admin. Utiliser `/sylius:menu` pour ajouter l'entrée.
- **Cache oublié** : les grids sont compilées dans le container — toute modif YAML / listener / `AbstractGrid` exige `symfony console cache:clear`. En dev, `APP_ENV=dev` recompile souvent, mais les listeners tagués nécessitent toujours un clear.
- **Modifier un template de grid vendor** : patch volatile (saute au prochain `composer update`). Pour un rendu custom, utiliser `TwigField` avec un template propre plutôt qu'override de vendor.

## Argument optionnel

`/sylius:grid sylius_admin_product_review --disable=title,filters.title` — désactive le champ `title` et le filtre `title` sur le grid des reviews.

`/sylius:grid sylius_admin_product --listener=variantSelectionMethod` — génère un `AdminProductsGridListener` qui ajoute `variantSelectionMethod`.

`/sylius:grid app_admin_supplier --new=Supplier` — scaffolde une nouvelle grid `AbstractGrid` sur la resource `Supplier`.

`/sylius:grid` sans argument — demande la grid cible et l'opération (disable/add/reorder/new) puis guide pas à pas.

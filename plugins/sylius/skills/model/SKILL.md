---
name: model
description: Étend un modèle Sylius natif (Customer, Country, ShippingMethod…) : sous-classe `Sylius\Component\*\Model\Base*`, déclare sous `sylius_<bundle>.resources.<r>.classes.model`, génère la migration Doctrine. Cas régulier et translatable.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /model — Customiser un modèle Sylius

Tu aides à **étendre un modèle Sylius natif** pour lui ajouter des champs métier (ex. `flag` sur `Country`, `secondNumber` sur `Customer`, `icon` sur `PaymentMethod`, `estimatedDeliveryTime` sur `ShippingMethod`). Le pattern officiel consiste à créer une sous-classe dans `src/Entity/` qui étend la classe Sylius, puis à rebrancher la resource via `config/packages/_sylius.yaml` pour que repositories, factories, controllers et grids utilisent la classe customisée.

Référence officielle : [docs.sylius.com/the-customization-guide/customizing-models](https://docs.sylius.com/the-customization-guide/customizing-models).

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine.
2. Vérifier `sylius/sylius` (ou le bundle concerné, ex. `sylius/addressing-bundle`) dans les dépendances.
   - Présent → OK.
   - Absent → *« Ce skill cible Sylius (customization de modèle via `sylius_<bundle>.resources…classes.model`). Je ne trouve pas `sylius/sylius`. On continue quand même ? »*
3. Si le modèle à étendre est **translatable** (champs multilingues) → basculer sur la skill **`/sylius:translation-entity`** qui gère la paire `Entity` + `EntityTranslation` en détail, puis revenir ici pour la partie config de resource.

## Règles fondamentales

- **Étendre la classe Core quand elle existe** : beaucoup de modèles du namespace `Sylius\Component\*\Model` sont déjà étendus dans `Sylius\Component\Core\Model`. Toujours partir de la version Core si elle existe (`Sylius\Component\Core\Model\Customer`, pas `Sylius\Component\Customer\Model\Customer`) — sinon on perd les champs Core (`user`, `orders`, etc.).
- **Implémenter l'interface correspondante** : `class Country extends BaseCountry implements CountryInterface` — même si ça paraît redondant, ça permet aux type hints Sylius (`CountryInterface`) de matcher la classe customisée. Sans ça, Doctrine charge la classe mais les services typés sur l'interface peuvent mal résoudre.
- **Table héritée, pas recréée** : on garde `@ORM\Table(name: 'sylius_country')` (ou le nom natif) pour **remplacer** le mapping natif, pas créer une table parallèle. Sylius détecte qu'on a redéclaré la resource et utilise notre classe sur la table existante.
- **Un bundle = une section YAML** : `Country` → `sylius_addressing`, `Customer` → `sylius_customer`, `ShippingMethod` → `sylius_shipping`, `PaymentMethod` → `sylius_payment`, `Product` → `sylius_product`, etc. Le mapping bundle ↔ resource n'est pas uniforme — toujours vérifier le nom exact du bundle via `grep -r "sylius_<guess>:" vendor/sylius/sylius/src/Sylius/Bundle/` ou la doc du bundle.
- **Ne pas toucher aux repositories/factories/controllers** sauf besoin spécifique — la resource config suffit à ce que Sylius les câble sur la nouvelle classe.
- **Migration obligatoire** : toute modification du schéma passe par `doctrine:migrations:diff` + `migrations:migrate`. `doctrine:schema:update --force` est interdit en prod.
- **Xdebug / cache** : après modif de `_sylius.yaml`, toujours `symfony console cache:clear` sinon la resource reste mappée sur l'ancienne classe.

## Déroulement

### 1 — Cadrer le besoin

Demander (ou confirmer si déjà fourni) :

- **Modèle cible** à customiser : `Country`, `Customer`, `ShippingMethod`, `PaymentMethod`, `Product`, `Taxon`, `Address`, `Order`, etc.
- **Champ à ajouter** : nom, type Doctrine (`string`, `integer`, `boolean`, `text`, `datetime`, `decimal`…), nullable ou non, valeur par défaut.
- **Translatable ?** → si oui, rediriger vers `/sylius:translation-entity` pour la paire `Model` / `ModelTranslation`, puis revenir pour la config resource.
- **Héritage** : le modèle existe-t-il dans `Sylius\Component\Core\Model` ? Si oui, partir de là. Sinon, partir de `Sylius\Component\<Bundle>\Model`.

### 2 — Repérer la classe à étendre

```bash
# Vérifier l'existence de la version Core
ls vendor/sylius/sylius/src/Sylius/Component/Core/Model/<ModelName>.php 2>/dev/null

# Sinon, base component
ls vendor/sylius/sylius/src/Sylius/Component/*/Model/<ModelName>.php 2>/dev/null
```

Noter le **namespace complet** et l'**interface associée** (presque toujours `<ModelName>Interface` dans le même namespace).

### 3 — Créer la classe customisée

Exemple pour `Country` + champ `flag` — `src/Entity/Addressing/Country.php` :

```php
<?php

declare(strict_types=1);

namespace App\Entity\Addressing;

use Doctrine\ORM\Mapping as ORM;
use Sylius\Component\Addressing\Model\Country as BaseCountry;
use Sylius\Component\Addressing\Model\CountryInterface;

#[ORM\Entity]
#[ORM\Table(name: 'sylius_country')]
class Country extends BaseCountry implements CountryInterface
{
    #[ORM\Column(type: 'string', nullable: true)]
    private ?string $flag = null;

    public function getFlag(): ?string
    {
        return $this->flag;
    }

    public function setFlag(?string $flag): void
    {
        $this->flag = $flag;
    }
}
```

Points de vigilance :

- **Attributs PHP 8 (`#[ORM\…]`) plutôt qu'annotations docblock** — c'est le standard depuis Sylius 2.x et Symfony 6.x. Les anciennes annotations `/** @ORM\Entity */` fonctionnent encore mais sont en extinction.
- **Propriété `private` + getter/setter explicites** : pas de `public` sur les colonnes Doctrine, et pas de `protected` non plus (ça complique les tests).
- **Typage strict** : `declare(strict_types=1)` en tête, getters/setters avec types de retour et de paramètres.
- **Valeur par défaut en PHP** (`?string $flag = null`) quand `nullable: true` — ça évite les `uninitialized property` à l'instanciation.

### 4 — Déclarer la resource customisée

Dans `config/packages/_sylius.yaml`, sous la section du bundle concerné :

```yaml
sylius_addressing:
    resources:
        country:
            classes:
                model: App\Entity\Addressing\Country
```

Table de correspondance bundle ↔ modèles les plus fréquents :

| Modèle                | Section YAML         | Resource key       |
| --------------------- | -------------------- | ------------------ |
| `Country`             | `sylius_addressing`  | `country`          |
| `Province`            | `sylius_addressing`  | `province`         |
| `Zone`                | `sylius_addressing`  | `zone`             |
| `Address`             | `sylius_addressing`  | `address`          |
| `Customer`            | `sylius_customer`    | `customer`         |
| `CustomerGroup`       | `sylius_customer`    | `customer_group`   |
| `ShopUser`            | `sylius_shop_user`   | `shop_user`        |
| `AdminUser`           | `sylius_admin_user`  | `admin_user`       |
| `Product`             | `sylius_product`     | `product`          |
| `ProductVariant`      | `sylius_product`     | `product_variant`  |
| `Taxon`               | `sylius_taxonomy`    | `taxon`            |
| `ShippingMethod`      | `sylius_shipping`    | `shipping_method`  |
| `PaymentMethod`       | `sylius_payment`     | `payment_method`   |
| `Order`               | `sylius_order`       | `order`            |
| `OrderItem`           | `sylius_order`       | `order_item`       |
| `Promotion`           | `sylius_promotion`   | `promotion`        |
| `Channel`             | `sylius_channel`     | `channel`          |
| `Currency`            | `sylius_currency`    | `currency`         |
| `Locale`              | `sylius_locale`      | `locale`           |

En cas de doute, `grep -rn "sylius_resource" vendor/sylius/sylius/src/Sylius/Bundle/<BundleName>/Resources/config/app/` donne la section à utiliser.

### 5 — Cas translatable (rappel)

Si le modèle customisé est déjà traduisible (ex. `ShippingMethod`, `Taxon`, `Product`) **et** qu'on veut ajouter un champ **non traduisible**, le pattern reste celui du §3/§4.

Si on veut ajouter un champ **traduisible** (qui doit varier selon la locale), la config YAML doit aussi déclarer la translation :

```yaml
sylius_shipping:
    resources:
        shipping_method:
            classes:
                model: App\Entity\Shipping\ShippingMethod
            translation:
                classes:
                    model: App\Entity\Shipping\ShippingMethodTranslation
```

Pour générer correctement la paire `ShippingMethod` + `ShippingMethodTranslation`, utiliser **`/sylius:translation-entity`**.

### 6 — Migration

```bash
symfony console doctrine:migrations:diff
```

Inspecter le fichier généré dans `migrations/` :

- Doit contenir un `ALTER TABLE sylius_<table> ADD <colonne>` — **pas** un `CREATE TABLE` (sinon le mapping extends est mal câblé, en général un oubli de `@ORM\Table(name: ...)` ou d'interface).
- Vérifier qu'aucune colonne existante n'est droppée par erreur (faux positif classique quand le mapping vendor a dérivé).

Puis :

```bash
symfony console doctrine:migrations:migrate
```

Pour cadrer la migration proprement, enchaîner avec **`/symfony:doctrine-migration`**.

### 7 — Cache et vérification

```bash
symfony console cache:clear
symfony console doctrine:schema:validate
```

`schema:validate` doit retourner *"The mapping files are correct"* et *"The database schema is in sync with the mapping files"* — si l'un des deux échoue, c'est typiquement que :

- La resource n'est pas déclarée sous la bonne section YAML.
- L'interface (`CountryInterface`) n'est pas implémentée → Doctrine charge deux fois la même entité.
- Le nom de table ne correspond pas à celui du parent.

### 8 — Exposer le champ dans l'admin (optionnel)

Si le champ doit être éditable, étendre le `FormType` natif via un `FormTypeExtension` taguée `form.type_extension`, puis pousser un template dans le bon Twig Hook pour qu'il soit réellement rendu. Pour cadrer proprement l'ensemble (distinction `sylius_shop.*` vs `sylius_admin.*`, priorités, hooks, champs dynamiques), enchaîner avec **`/sylius:form`**.

### 9 — Clôture

Afficher :

- **Fichiers créés/modifiés** : `src/Entity/<Bundle>/<Model>.php`, `config/packages/_sylius.yaml`, éventuellement `src/Form/Extension/…`.
- **Migration générée** + commande de migrate.
- **Ce qui reste à faire** : form extension pour l'admin/shop, grid customization si le champ doit apparaître en listing, éventuellement fixtures pour les tests.

## Pièges fréquents

- **Oublier la version Core** : étendre `Sylius\Component\Customer\Model\Customer` alors que `Sylius\Component\Core\Model\Customer` existe → tu perds la relation vers `User`, les commandes, etc. Toujours vérifier d'abord l'existence dans `Core`.
- **`sylius_<bundle>` mal orthographié** : `sylius_shipping` et non `sylius_shipping_method`. La section YAML est le nom du **bundle**, pas de la resource. Un YAML mal nommé est silencieusement ignoré → la classe n'est jamais prise en compte.
- **`CREATE TABLE` dans la migration** : signe que Doctrine voit la nouvelle entité comme un type distinct. Causes : pas d'`extends BaseXxx`, pas d'`implements XxxInterface`, ou `@ORM\Table(name: ...)` oublié/différent du parent.
- **Champ translatable mis sur la mauvaise classe** : si le contenu doit varier par locale, il va sur `*Translation`, pas sur l'entité principale. Sinon, `WHERE name = ...` ne se comporte pas comme attendu en contexte multi-locale.
- **Cache pas vidé** après modif de `_sylius.yaml` : la resource reste mappée sur l'ancienne classe, les repositories renvoient des instances de `BaseCountry`, pas de `App\Entity\Addressing\Country`. Toujours `cache:clear` après une modif de resource.
- **Modifier la classe Core directement** dans `vendor/` : ne jamais patcher `vendor/sylius/sylius/` — les modifs sautent au prochain `composer update`. Toujours passer par l'extension dans `src/Entity/`.
- **Sylius 2.x / attributs** : si le projet est sur Sylius 1.x, les annotations docblock `/** @ORM\Entity */` sont requises. Sur Sylius 2.x, préférer les attributs PHP 8 `#[ORM\Entity]`. Mélanger les deux dans la même classe marche, mais est source de confusion.

## Argument optionnel

`/sylius:model Country flag:string` — cadre l'extension de `Country` avec un champ `flag` de type string.

`/sylius:model Customer secondNumber:string?` — ajoute un champ nullable `secondNumber` à `Customer`.

`/sylius:model ShippingMethod estimatedDeliveryTime:string` — gère `ShippingMethod` (Core), champ non traduisible.

`/sylius:model` sans argument — demande le modèle cible et le(s) champ(s) à ajouter, puis détecte automatiquement la section YAML et la classe Core à étendre.

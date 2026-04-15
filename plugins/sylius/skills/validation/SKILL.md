---
name: validation
description: Customise la validation d'un resource Sylius : `config/validator/<Model>.yaml` + groupe custom rebranché via `sylius.form.type.*.validation_groups`. PromotionRule/Action via ChannelCodeCollection. Champ absent → `/sylius:model`.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /validation — Customiser la validation Sylius

Tu aides à **redéfinir les contraintes de validation** d'un resource Sylius (longueur mini d'un champ, plage de montants, règles métier) **sans patcher le vendor**. Le pattern officiel : écrire un fichier `config/validator/<Model>.yaml|xml` qui override les contraintes dans un **nouveau groupe de validation** (ex. `app_product`), puis rebrancher ce groupe via `sylius.form.type.*.validation_groups` dans `config/services.yaml`. Pour `ShippingMethodRule`, `PromotionRule`, `PromotionAction`, `PromotionCoupon`, `CatalogPromotionAction`, `CatalogPromotionScope`, `ZoneMember`, la logique passe par `ChannelCodeCollection` + un paramètre indexé par la **clé de règle**.

Référence officielle : [docs.sylius.com/the-customization-guide/customizing-validation](https://docs.sylius.com/the-customization-guide/customizing-validation).

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine.
2. Vérifier `sylius/sylius` dans les dépendances.
   - Présent → OK.
   - Absent → *« Ce skill cible Sylius (override validation via groupes `sylius.form.type.*.validation_groups`). Je ne trouve pas `sylius/sylius`. On continue quand même ? »*
3. Si la demande concerne un champ qui **n'existe pas encore** sur le modèle (ex. valider un `secondaryPhoneNumber` jamais ajouté) → basculer sur **`/sylius:model`** pour créer le champ + migration, puis revenir ici pour poser la contrainte.
4. Si la demande est *« ajouter une contrainte `#[Assert\*]` directement en attribut PHP sur une entité custom »* (pas une entité Sylius native) → basculer sur **`/symfony:validation-constraints`** : la stratégie n'est pas de créer un groupe Sylius parallèle, c'est de valider l'entité applicative directement.

## Règles fondamentales

- **Le groupe par défaut Sylius est `sylius`** : toutes les resources sont validées dans ce groupe. L'override consiste à **créer un nouveau groupe** (ex. `app_product`) et à remplacer `[sylius]` par `[app_product]` dans le paramètre `sylius.form.type.*.validation_groups`. **Ne jamais redéfinir le groupe `sylius` lui-même** — ça casse la validation des chemins du Core qui comptent encore dessus.
- **Ton fichier `config/validator/` override le vendor, il ne l'étend pas** : Symfony merge `class` + `property`, mais **à l'intérieur d'une property, la dernière déclaration gagne par constraint**. Concrètement, si tu ne redéclares que `Length`, `NotBlank` hérité du vendor reste actif, mais si tu redéclares `Length` et oublies le `min`, la vendor version n'est **pas** utilisée en fallback — il faut repartir du fichier vendor comme base. Sylius fournit les sources : [vendor XML reference](https://github.com/Sylius/Sylius/tree/v2.0.7/src/Sylius/Bundle/ProductBundle/Resources/config/validation).
- **Un champ translatable se valide sur la `*Translation`**, pas sur la classe mère : `name` sur `Product` vit en réalité sur `ProductTranslation`. Le fichier cible est `config/validator/ProductTranslation.yaml` et l'override passe par **deux** paramètres :
  ```yaml
  sylius.form.type.product_translation.validation_groups: [app_product]
  sylius.form.type.product.validation_groups: [app_product]   # propagation vers le form parent
  ```
  Sans le second, la `Product` reste en `sylius` et ne propage pas le nouveau groupe → les form children n'utilisent pas tes contraintes.
- **Cas spéciaux (rules / actions / zones)** : `ShippingMethodRule`, `ShippingMethod`, `PromotionRule`, `PromotionAction`, `PromotionCoupon`, `CatalogPromotionAction`, `CatalogPromotionScope`, `ZoneMember` ne passent **pas** par le paramètre classique `sylius.form.type.*.validation_groups`. Leur configuration est un **tableau imbriqué** sur la propriété `configuration`, donc la validation passe obligatoirement par `Sylius\Bundle\CoreBundle\Validator\Constraints\ChannelCodeCollection` + `Collection`, et le paramètre de rebranchement est **indexé par la clé de rule** (ex. `sylius.shipping.shipping_method_rule.validation_groups.order_total_greater_than_or_equal: [app_...]`). Chaque type de rule (`order_total_less_than_or_equal`, etc.) demande **son propre** groupe — pas de wildcard.
- **`allowExtraFields: true`** sur `Collection` dans le cas `configuration` : indispensable, sinon Symfony remonte une erreur dès qu'un champ non listé apparaît (et plusieurs clés dynamiques existent selon le contexte).
- **Les messages de validation sont des clés de traduction** (`sylius.product.name.min_length`, `app.product.name.min_length`) : toujours déclarer une entrée dans `translations/messages.<locale>.yaml` (domaine `messages` ou `validators`), sinon l'UI affiche la clé brute. Voir [Symfony validation translations](https://symfony.com/doc/current/validation/translations.html).
- **Cache obligatoire après toute modif** de `config/validator/*`, `config/services.yaml`, `config/packages/_sylius.yaml` : le container et le mapping de validation sont compilés. `symfony console cache:clear` systématique, sinon l'ancien groupe reste actif et le debug devient trompeur.
- **Les `assert` attributes PHP sur une entité Sylius étendue** (ex. un `App\Entity\Product extends BaseProduct` avec `#[Assert\Length]`) sont **fusionnés** avec le XML du vendor — mais l'ordre de merge n'est pas garanti, c'est source de bugs difficiles à tracer. Préférer la stratégie `config/validator/*.yaml|xml` + nouveau groupe, qui découple totalement l'override du cycle de vie de l'entité.

## Déroulement

### 1 — Cadrer le besoin

Demander (ou confirmer si déjà fourni) :

- **Resource cible** : `Product`, `ProductTranslation`, `Customer`, `ShippingMethodRule`, `PromotionRule`, `PromotionCoupon`, `Address`, `Channel`, etc.
- **Champ + contrainte** : `name` → min 10 / max 255 ; `amount` → range 1000-1000000 ; `code` → `NotBlank` custom ; etc.
- **Portée** : contrainte sur un form classique (shop / admin), ou sur une `*Rule` / `*Action` (clé dynamique de configuration).
- **Nom du groupe custom** : convention `app_<resource>` (ex. `app_product`, `app_customer`, `app_shipping_method_rule_order_grater_than_or_equal`). Choisis un nom stable, il est cité dans plusieurs endroits.
- **Locale(s) des messages** : pour prévoir les clés de traduction.

### 2 — Repérer le fichier vendor de référence

Prendre la version exacte installée (ne pas taper au hasard sur `master`) :

```bash
# Version installée
composer show sylius/sylius | grep -i '^versions'

# Localiser le XML vendor pour la resource
ls vendor/sylius/sylius/src/Sylius/Bundle/*/Resources/config/validation/ | grep -i <Model>
# ex : ProductBundle/Resources/config/validation/ProductTranslation.xml
```

Ouvrir ce fichier : il liste **toutes** les contraintes et les groupes (`sylius`, parfois `sylius_*`). Sert de **base de copie** à adapter. Ne jamais le modifier en place dans `vendor/`.

### 3 — Cas standard : champ sur un resource classique

Exemple : forcer `min: 10` sur `Product.name` (stocké dans `ProductTranslation.name`).

#### a. Créer `config/validator/ProductTranslation.yaml`

```yaml
# config/validator/ProductTranslation.yaml

Sylius\Component\Product\Model\ProductTranslation:
    properties:
        name:
            - NotBlank:
                message: sylius.product.name.not_blank
                groups: [app_product]
            - Length:
                min: 10
                minMessage: sylius.product.name.min_length
                max: 255
                maxMessage: sylius.product.name.max_length
                groups: [app_product]
```

Équivalent XML si la team utilise exclusivement XML :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<constraint-mapping xmlns="http://symfony.com/schema/dic/constraint-mapping"
                    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                    xsi:schemaLocation="http://symfony.com/schema/dic/constraint-mapping http://symfony.com/schema/dic/services/constraint-mapping-1.0.xsd">
    <class name="Sylius\Component\Product\Model\ProductTranslation">
        <property name="name">
            <constraint name="NotBlank">
                <option name="message">sylius.product.name.not_blank</option>
                <option name="groups">app_product</option>
            </constraint>
            <constraint name="Length">
                <option name="min">10</option>
                <option name="minMessage">sylius.product.name.min_length</option>
                <option name="max">255</option>
                <option name="maxMessage">sylius.product.name.max_length</option>
                <option name="groups">app_product</option>
            </constraint>
        </property>
    </class>
</constraint-mapping>
```

Points de vigilance :

- Toutes les contraintes de la property doivent pointer sur `groups: [app_product]` — si une contrainte reste sur `[sylius]`, elle ne sera plus appliquée une fois le form rebranché sur `app_product`.
- Si tu conserves certaines contraintes du vendor telles quelles, **copie-les aussi** dans ton YAML avec le nouveau groupe. Sinon elles disparaissent lors du passage au groupe custom.

#### b. Rebrancher le form sur le nouveau groupe

`config/services.yaml` :

```yaml
parameters:
    sylius.form.type.product_translation.validation_groups: [app_product]
    sylius.form.type.product.validation_groups: [app_product]  # parent propage l'enfant translation
```

Pour trouver le **nom exact** du paramètre à override :

```bash
symfony console debug:container --parameters --env=dev | grep validation_groups
```

La sortie liste tous les paramètres de la forme `sylius.form.type.<resource>.validation_groups` — copie-colle le bon nom, sans deviner (la convention n'est pas 100 % régulière : `product_translation`, pas `product.translation`).

#### c. Vérifier

```bash
symfony console cache:clear
symfony console debug:container --parameters | grep app_product
```

Soumettre un form admin avec un `name` de 5 caractères : l'erreur `sylius.product.name.min_length` doit apparaître.

### 4 — Cas spécial : `ShippingMethodRule`, `PromotionRule`, etc.

Exemple : imposer un montant minimum de 10 € (1000 centimes) pour la rule `order_total_greater_than_or_equal`.

#### a. Créer `config/validator/ShippingMethodRule.yaml`

```yaml
# config/validator/ShippingMethodRule.yaml

Sylius\Component\Shipping\Model\ShippingMethodRule:
    properties:
        configuration:
            - Sylius\Bundle\CoreBundle\Validator\Constraints\ChannelCodeCollection:
                  groups: app_shipping_method_rule_order_grater_than_or_equal
                  validateAgainstAllChannels: true
                  channelAwarePropertyPath: shippingMethod
                  constraints:
                      - Collection:
                            fields:
                                amount:
                                    - range:
                                          groups: app_shipping_method_rule_order_grater_than_or_equal
                                          min: 1000
                                          max: 1000000
                  allowExtraFields: true
```

Points de vigilance :

- `ChannelCodeCollection` wraps **obligatoirement** le `Collection` dès qu'une règle est `ChannelAware` (c'est le cas de toutes les rules Sylius). Sans ce wrapping, la contrainte ne s'applique qu'au canal par défaut.
- `channelAwarePropertyPath: shippingMethod` pointe vers la property parente qui expose les channels (ici, `ShippingMethodRule::$shippingMethod`). Pour une `PromotionRule`, c'est `promotion`. Pour `CatalogPromotionAction`, c'est `catalogPromotion`.
- `allowExtraFields: true` est non-négociable : les payloads `configuration` comportent des clés variables selon le type de rule.
- Les valeurs de `min/max` sont **en centimes** (toute somme monétaire dans Sylius est stockée en plus petite unité).

#### b. Rebrancher via un paramètre indexé par la clé de rule

`config/services.yaml` :

```yaml
parameters:
    sylius.shipping.shipping_method_rule.validation_groups:
        order_total_greater_than_or_equal: [app_shipping_method_rule_order_grater_than_or_equal]
```

- La clé `order_total_greater_than_or_equal` **doit exactement matcher** la `rule_key` enregistrée dans le service container. Chercher la liste complète :
  ```bash
  symfony console debug:container --tag=sylius.shipping_method.rule_checker
  ```
- Pour couvrir `order_total_less_than_or_equal`, **ajouter une entrée séparée** avec son propre groupe, son propre fichier de validator. Pas de wildcard possible.
- Équivalent pour les promos :
  - `sylius.promotion.promotion_rule.validation_groups.<rule_key>`
  - `sylius.promotion.promotion_action.validation_groups.<action_key>`
  - `sylius.promotion.catalog_promotion_action.validation_groups.<action_key>`
  - `sylius.promotion.catalog_promotion_scope.validation_groups.<scope_key>`

#### c. Vérifier depuis l'admin

Créer une `ShippingMethod` avec la rule `order_total_greater_than_or_equal` et un montant `amount: 500` (5 €) → l'admin doit rejeter avec l'erreur range. Montant `1500` → pass.

### 5 — Messages de validation : traductions

Chaque clé utilisée dans `message` / `minMessage` / `maxMessage` doit exister côté translations :

```yaml
# translations/validators.en.yaml
sylius:
    product:
        name:
            not_blank: Please enter product name.
            min_length: Product name must be at least {{ limit }} characters long.
            max_length: Product name cannot be longer than {{ limit }} characters.
```

Domaine : `validators` est le domaine natif Symfony pour les contraintes de validation. Si tu utilises le domaine `messages` (par défaut), déclarer-les dans `messages.<locale>.yaml` et utiliser `messageTemplate` + `domain: messages` dans la contrainte.

Éviter le piège classique : si la clé existait déjà côté Sylius vendor (comme `sylius.product.name.min_length`) mais que le paramètre de message vendor attend un placeholder `{{ limit }}` différent, le template ne remplacera pas la variable — préfixer les clés custom avec `app.` évite tout chevauchement.

### 6 — Group Sequence Validation (avancé)

Quand plusieurs contraintes doivent s'exécuter **dans un ordre précis** (valider `code` avant `name`, par exemple), Symfony supporte le group sequence provider. Pour que Sylius l'active, le form doit rester en validation group `[Default]` :

```php
// src/Entity/Product.php
use Symfony\Component\Validator\GroupSequenceProviderInterface;

class Product extends BaseProduct implements GroupSequenceProviderInterface
{
    public function getGroupSequence(): array
    {
        return ['Product', 'app_product_strict'];
    }
}
```

Côté `config/services.yaml`, laisser :

```yaml
parameters:
    sylius.form.type.product.validation_groups: [Default]
```

Réf : [Symfony — Group Sequence Provider](https://symfony.com/doc/current/validation/sequence_provider.html). Pas évident à débugger : activer seulement si la team en a réellement besoin.

### 7 — Vérification finale

```bash
symfony console cache:clear
symfony console debug:container --parameters | grep -E '(app_product|app_shipping)'
```

Tester le parcours qui déclenche la validation :

- Form shop/admin : saisir une valeur invalide → message attendu s'affiche.
- Rule (shipping / promotion) : configurer un `amount` hors range depuis l'admin → l'admin refuse la soumission.
- API : `POST` / `PUT` sur la ressource concernée → réponse `422` avec la violation bien taggée.

Si la contrainte ne se déclenche pas :

- le form n'envoie pas le bon `validation_groups` → vérifier `sylius.form.type.*.validation_groups` ;
- la clé de rule ne matche pas → vérifier la liste des `rule_checker` / `action_checker` ;
- la contrainte tombe encore dans le groupe `sylius` → un `groups: [...]` a été oublié dans le YAML ;
- la translation manque → on voit la clé `sylius.product.name.min_length` brute au lieu du texte.

### 8 — Clôture

Afficher :

- **Fichiers créés/modifiés** : `config/validator/<Model>.yaml`, `config/services.yaml` (paramètres `validation_groups`), `translations/validators.<locale>.yaml`.
- **Commandes de vérif** : `debug:container --parameters`, `debug:container --tag=sylius.*.rule_checker`, `cache:clear`.
- **Ce qui reste à faire** : couvrir les autres rule keys si la contrainte doit être uniforme, ajouter les traductions pour toutes les locales actives, tests fonctionnels (form submit + API `422`).

## Pièges fréquents

- **Override partiel** : ne redéclarer que `Length` dans le YAML en pensant garder `NotBlank` du vendor. Une fois le group basculé sur `app_product`, plus rien du vendor ne s'applique — `NotBlank` doit être recopié.
- **Oublier `sylius.form.type.product.validation_groups`** quand le champ est sur `ProductTranslation` : le form parent continue d'émettre le groupe `sylius`, donc tes contraintes custom ne sont jamais validées.
- **Rule key fausse** : `order_total_greater_or_equal` au lieu de `order_total_greater_than_or_equal`. Le paramètre est ignoré silencieusement, la contrainte retombe sur la default. Toujours lister via `debug:container --tag=sylius.shipping_method.rule_checker`.
- **Oublier `allowExtraFields: true`** sur `Collection` dans `configuration` : le premier champ imprévu fait exploser la validation complète de la rule.
- **Montants en euros au lieu de centimes** dans `min/max` d'une `range` : `min: 10` laisse passer tout ≥ 10 centimes. Tous les montants Sylius sont en centimes → multiplier par 100.
- **Modifier le fichier `vendor/sylius/…/validation/*.xml`** directement : saute au `composer update` suivant, casse les autres projets qui partagent le vendor. Toujours créer le fichier côté `config/validator/`.
- **`#[Assert\*]` en attribut PHP** sur un override d'entité + fichier XML vendor qui a encore le même champ : le merge Symfony fusionne les contraintes, les erreurs se démultiplient (deux messages NotBlank au lieu d'un). Choisir **un seul** endroit (YAML/XML recommandé pour rester alignés avec la doc Sylius).
- **Groupe `[Default]` attendu par `getGroupSequence()` mais paramètre laissé à `[app_product]`** : `getGroupSequence()` n'est jamais appelé (condition d'entrée non satisfaite). Lire la doc Symfony sur le sujet avant d'activer le pattern.
- **Cache pas vidé** : la modif du YAML ou du paramètre ne prend pas effet. `cache:clear` systématique sur `dev` **et** `prod` après un déploiement.

## Argument optionnel

`/sylius:validation ProductTranslation name min=10 max=255` — override la longueur de `Product.name` via `ProductTranslation`, groupe auto `app_product`.

`/sylius:validation ShippingMethodRule order_total_greater_than_or_equal amount min=1000 max=1000000` — contrainte range sur la rule `order_total_greater_than_or_equal`, groupe auto `app_shipping_method_rule_order_grater_than_or_equal`.

`/sylius:validation PromotionCoupon code pattern=/^[A-Z0-9]{6,12}$/` — contrainte regex sur le code coupon via `PromotionCoupon`.

`/sylius:validation` sans argument — demande la resource cible, le champ, les contraintes, et guide pas à pas (standard ou cas spécial rule/action).

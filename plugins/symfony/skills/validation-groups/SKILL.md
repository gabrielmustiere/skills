---
name: validation-groups
description: Groupes de validation Symfony — `GroupSequence`, validation conditionnelle (`When`, `Expression`, `AtLeastOneOf`). Déclenche sur "groupes validation", "validation_groups", "GroupSequence". Impose partage explicite forms/validator.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /validation-groups — Groupes, séquences, validations conditionnelles

> **Utilise quand** tu organises plusieurs scénarios de validation (`GroupSequence`, `When`, `Expression`).
> **Pas quand** tu poses la contrainte de base → `/symfony:validation-constraints`.
> **Pas quand** tu appelles le Validator hors form → `/symfony:validation-use`.

Tu structures la validation d'une classe qui a plusieurs scénarios (création vs édition, draft vs publish, API vs back-office). Tu évites l'enfer des `if` dans des `Callback` en exprimant les règles sous forme de **groupes** + **contraintes conditionnelles** déclaratives.

## Détection préalable

1. Lire `composer.json`, vérifier `symfony/validator`.
2. Si `symfony/form` présent → les forms passent par l'option `validation_groups` (de `configureOptions`). Ces groupes doivent matcher ce que tu déclares côté entité.
3. Si `sylius/sylius` présent → groupes par défaut des forms Sylius : `['Default', 'sylius']`. Toute contrainte custom doit porter ces deux groupes (ou un surensemble).

## Quand un groupe est justifié

Pose-toi la question avant d'ajouter des groupes : **la règle change-t-elle selon le contexte ?**

- Oui (SKU requis à l'édition, optionnel à la création ; mot de passe requis à l'inscription, optionnel au changement d'email) → groupe.
- Non (un email doit toujours être un email) → pas de groupe, laisser dans `Default`.

Les groupes sont un outil puissant mais ils fragmentent la lisibilité. **Un ou deux groupes custom suffisent** dans la plupart des classes. Au-delà, reconsidérer le design (DTO dédié par scénario vs entité unique surchargée).

## Règles fondamentales

- **`Default` est toujours implicite**. Une contrainte sans `groups` appartient au groupe `Default`.
- **`groups: ['Default', 'creation']`** → la contrainte participe aux deux groupes. Utile pour une règle qui doit passer partout mais aussi dans un scénario spécifique.
- **Valider un groupe ≠ valider `Default`**. `$validator->validate($obj, null, ['creation'])` ne valide **que** les contraintes du groupe `creation`. Pour joindre `Default`, passer `['Default', 'creation']` explicitement.
- **Groupes côté form : `configureOptions`**, pas `createForm`. L'option peut être un tableau statique, une closure (dynamique selon l'objet), ou un `GroupSequence`.
- **Nom de groupe = string stable**. Éviter `'creation_step_3'` qui décrit l'UI, préférer `'creation'`, `'publishing'`, `'archival'` (métier). Les noms de groupes sont du **contrat** entre l'entité, le form, les services.

## Déclaration des groupes sur contraintes

```php
use Symfony\Component\Validator\Constraints as Assert;

class Product
{
    #[Assert\NotBlank] // groupe Default
    private string $name;

    #[Assert\NotBlank(groups: ['publishing'])]
    #[Assert\Length(min: 20, groups: ['publishing'])]
    private ?string $description = null;

    #[Assert\NotBlank(groups: ['Default', 'creation'])]
    #[Assert\Regex(pattern: '/^[A-Z]{3}-\d{4}$/', groups: ['Default', 'creation'])]
    private ?string $sku = null;
}
```

Lecture : `description` n'est validée qu'à la publication ; `sku` est validé partout (`Default`) et explicitement au scénario `creation` (qui peut être appelé avec `['creation']` seul).

## Appel côté validator

```php
// Créer un produit : pas de description requise
$violations = $validator->validate($product, groups: ['Default', 'creation']);

// Publier un produit : description et règles standard
$violations = $validator->validate($product, groups: ['Default', 'publishing']);
```

## Appel côté form

### Statique

```php
public function configureOptions(OptionsResolver $resolver): void
{
    $resolver->setDefaults([
        'data_class' => Product::class,
        'validation_groups' => ['Default', 'creation'],
    ]);
}
```

### Dynamique (selon l'objet)

```php
$resolver->setDefaults([
    'data_class' => Product::class,
    'validation_groups' => function (FormInterface $form): array {
        $product = $form->getData();
        return $product->getId() === null
            ? ['Default', 'creation']
            : ['Default', 'edition'];
    },
]);
```

### Séquence (court-circuit)

Voir plus bas, `GroupSequence`.

## `GroupSequence` — validation en cascade

Valide les groupes dans l'ordre, **s'arrête au premier échec**. Utile pour ne pas afficher 10 erreurs alors que la structure de base est cassée.

```php
use Symfony\Component\Validator\Constraints as Assert;

#[Assert\GroupSequence(['Product', 'strict'])]
class Product
{
    #[Assert\NotBlank] // groupe 'Product' (implicite quand GroupSequence est sur la classe)
    private string $name;

    #[Assert\Length(min: 20, groups: ['strict'])]
    private string $description;
}
```

Lecture : si `name` est vide, `description` n'est même pas testée. `Default` devient un alias du groupe séquence.

Côté form :

```php
use Symfony\Component\Validator\Constraints\GroupSequence;

$resolver->setDefaults([
    'validation_groups' => new GroupSequence(['Product', 'strict']),
]);
```

**À éviter** : séquences > 3 groupes — ça cache des dépendances implicites.

## Validation conditionnelle déclarative

### `#[Assert\When]` (Symfony 6.2+)

Exécute des contraintes si une expression ExpressionLanguage évaluée sur l'objet est vraie.

```php
use Symfony\Component\Validator\Constraints as Assert;

class Order
{
    #[Assert\Choice(choices: ['pickup', 'delivery'])]
    private string $mode;

    #[Assert\When(
        expression: 'this.mode === "delivery"',
        constraints: [
            new Assert\NotBlank(message: 'order.address.required'),
            new Assert\Length(min: 5),
        ],
    )]
    private ?string $address = null;
}
```

Lisible, sans `Callback`, sans groupe artificiel. Premier recours pour une règle conditionnelle.

### `#[Assert\Expression]` (bloc complet)

Une seule règle, accès à l'objet et aux autres propriétés :

```php
#[Assert\Expression(
    expression: 'this.priceCents >= this.minPriceCents',
    message: 'product.price.below_minimum',
)]
class Product { /* ... */ }
```

### `#[Assert\Sequentially]` (court-circuit par propriété)

```php
#[Assert\Sequentially([
    new Assert\NotBlank(),
    new Assert\Length(min: 8),
    new Assert\NotCompromisedPassword(),
])]
private string $password;
```

Équivalent d'un `GroupSequence` mais à l'échelle d'une propriété. Si `NotBlank` échoue, `Length` n'est pas testé (évite deux messages).

### `#[Assert\AtLeastOneOf]`

Passe si **au moins une** des contraintes passe. Pour un champ qui accepte plusieurs formats.

```php
#[Assert\AtLeastOneOf([
    new Assert\Email(),
    new Assert\Regex(pattern: '/^\+?[0-9]{8,15}$/'),
], message: 'contact.invalid')]
private string $contact;
```

### `#[Assert\Compound]` — contrainte composite réutilisable

Pour factoriser un jeu de contraintes sous un seul attribut :

```php
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\Validator\Constraints\Compound;

#[\Attribute]
final class StrongPassword extends Compound
{
    protected function getConstraints(array $options): array
    {
        return [
            new Assert\NotBlank(),
            new Assert\Length(min: 12),
            new Assert\Regex(pattern: '/[A-Z]/', message: 'password.needs_uppercase'),
            new Assert\Regex(pattern: '/[0-9]/', message: 'password.needs_digit'),
            new Assert\NotCompromisedPassword(),
        ];
    }
}
```

Usage :

```php
#[StrongPassword]
private string $password;
```

Préférer `Compound` à une contrainte custom complète quand la règle est juste une agrégation. Contrainte custom (avec `Validator` dédié) uniquement si la logique dépasse la composition — appel à un service, règle dynamique.

## `ExtendsValidationFor` — contraintes sur classe tierce

Pour ajouter des contraintes à une classe de vendor que tu ne peux pas modifier (Sylius `Order`, n'importe quelle entité d'un bundle tiers).

```php
// src/Validation/ProductValidation.php
namespace App\Validation;

use Sylius\Component\Core\Model\Product;
use Symfony\Component\Validator\Attribute\ExtendsValidationFor;
use Symfony\Component\Validator\Constraints as Assert;

#[ExtendsValidationFor(Product::class)]
abstract class ProductValidation
{
    #[Assert\NotBlank(groups: ['Default', 'sylius'])]
    #[Assert\Length(min: 10, groups: ['Default', 'sylius'])]
    public string $name = '';
}
```

Règles :

- Classe `abstract` (jamais instanciée, sert de véhicule de métadonnées).
- Les propriétés déclarées doivent **exister** sur la classe cible, sinon `MappingException`.
- Les contraintes **s'ajoutent** à celles existantes, elles ne remplacent pas (comme l'héritage).
- Pour **retirer** une contrainte vendor : impossible par `ExtendsValidationFor`. Il faut override la resource (Sylius) ou remapper via `validator.yaml` (`auto_mapping` ciblé, config XML/YAML de haut niveau).

## Héritage et groupes

- Les contraintes d'un parent s'appliquent aux enfants dans leurs groupes d'origine.
- Pour différencier enfant/parent : mettre la contrainte parent dans un groupe spécifique (`groups: ['parent']`) et valider l'enfant avec un autre groupe.
- Le `Default` d'une classe inclut les contraintes `Default` de toutes les classes parentes sauf si la classe redéfinit `GroupSequenceProvider` / `GroupSequence`.

## Debug

```bash
symfony console debug:validator 'App\Entity\Product'
```

La colonne `Groups` liste les groupes de chaque contrainte. Permet de vérifier qu'une règle est bien dans `['Default', 'creation']` et pas seulement `['creation']`.

## Déroulement

### 1 — Cadrer les scénarios

Lister les contextes de validation distincts (création, publication, API admin, self-service…). Un scénario = un groupe (ou un tuple `['Default', 'X']`).

### 2 — Choisir l'outil

| Besoin                                                      | Outil                                         |
|-------------------------------------------------------------|-----------------------------------------------|
| Règle qui change selon le scénario                          | Groupes                                       |
| Règle conditionnelle sur une autre propriété                | `#[Assert\When]` ou `#[Assert\Expression]`    |
| Court-circuit sur une propriété                             | `#[Assert\Sequentially]`                      |
| Court-circuit sur la classe entière                         | `#[Assert\GroupSequence]`                     |
| Plusieurs formats acceptés                                  | `#[Assert\AtLeastOneOf]`                      |
| Règle composite réutilisable                                | `#[Assert\Compound]` (`extends Compound`)     |
| Règles sur classe vendor                                    | `#[ExtendsValidationFor]`                     |
| Règle métier complexe avec services                         | Contrainte custom + Validator (hors scope — doc Symfony "How to create a Custom Validation Constraint") |

### 3 — Câbler

- Côté entité : ajouter `groups` aux contraintes concernées, `Default` pour celles qui restent globales.
- Côté form : `validation_groups` dans `configureOptions` (statique ou closure).
- Côté service/API : `$validator->validate($obj, null, ['Default', 'scenario'])`.

### 4 — Vérifier

```bash
symfony console debug:validator 'App\Entity\...'
vendor/bin/phpstan analyse
```

Tester chaque scénario : un test par groupe qui vérifie les violations attendues et celles qui **ne doivent pas** apparaître.

### 5 — Clôture

Afficher :

- Scénarios identifiés et groupes correspondants.
- Fichiers modifiés (entité, form, service).
- Ce qui reste : tests par scénario, traductions si nouveaux messages, usage hors form (→ `/symfony:validation-use`), contraintes elles-mêmes (→ `/symfony:validation-constraints`).

## Delta Sylius

- Groupe `sylius` par défaut dans les forms du vendor. Toute contrainte custom destinée à être validée dans un form Sylius doit porter `groups: ['Default', 'sylius']`.
- Les resources Sylius peuvent override les forms dans `_sylius.yaml` → les options `validation_groups` sont résolues à ce niveau.
- Multi-channel : valider une entité channel-scopée implique parfois un groupe `channel_<code>` conditionnel — préférer un service dédié ou un `Callback` qui prend le channel en paramètre, pas un groupe par channel (explosion combinatoire).

## Argument optionnel

`/symfony:validation-groups Product creation,publishing` — structure l'entité `Product` pour ces deux scénarios et câble le form.

`/symfony:validation-groups src/Entity/Order.php` — audit des groupes d'une classe existante (incohérences entre entity et form, groupes orphelins).

`/symfony:validation-groups` sans argument — demande la classe et la liste des scénarios.

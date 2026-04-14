---
name: validation-constraints
description: Pose des contraintes Symfony via `#[Assert\*]` — entité/DTO/classe, catalogue (NotBlank, Email, UniqueEntity, Callback). Déclenche sur "contrainte Symfony", "Assert", "NotBlank", "UniqueEntity". Impose attributs PHP et auto-mapping.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /validation-constraints — Contraintes Symfony sur entité/DTO/classe

Tu poses ou révises les contraintes de validation d'une classe PHP (entité Doctrine, DTO API, value object). Tu utilises les **attributs PHP 8** `#[Assert\*]`, tu choisis la bonne cible (propriété, getter, classe), et tu ne duplique pas ce que Doctrine déduit déjà du mapping.

## Détection préalable (obligatoire)

1. Lire `composer.json`.
2. Vérifier `symfony/validator`.
   - Présent → OK.
   - Absent → `composer require symfony/validator` avant de continuer.
3. Si `symfony/form` présent → rappeler que les forms valident **automatiquement** la `data_class` après `handleRequest`. Pas de call manuel au validator dans le contrôleur (cf. `/symfony:form-handle`).
4. Si `sylius/sylius` présent → les contraintes custom doivent déclarer `groups: ['Default', 'sylius']` pour être prises par les forms Sylius (cf. `/symfony:validation-groups`).

## Règles fondamentales

- **Attributs PHP, pas YAML/XML** : la codebase est en attributs. YAML/XML n'est justifié que si l'entité vient d'un bundle tiers non-modifiable (auquel cas → `ExtendsValidationFor`, cf. `/symfony:validation-groups`).
- **Validation auto-mappée Doctrine déjà en place** : `nullable: false` → `NotNull`, `unique: true` → `UniqueEntity` (à confirmer manuellement, cf. plus bas), `length: N` → `Length(max: N)`. Ne pas redéclarer ces contraintes sauf surcharge métier (message custom, `groups`).
- **Une contrainte = une intention métier**. `#[Assert\NotBlank]` sur un champ déjà `nullable: false` n'apporte rien à la persistance, mais clarifie le contrat form/API (le validator accepte `''` pour `NotNull`, refuse pour `NotBlank`).
- **Messages ICU** : `message: 'product.name.blank'` et traduction dans `translations/validators.<locale>.yaml`. Pas de messages en clair dans le code, sauf prototype jetable.
- **Ne jamais mettre la logique métier dans un `Callback`** si un service peut s'en charger (via `Expression` + `ExpressionLanguage`, ou une contrainte custom — cf. `/symfony:validation-groups`).
- **Cibles par défaut : la propriété**. Getter uniquement si la valeur est calculée (`isPasswordSafe()`). Classe uniquement si la contrainte dépend de plusieurs propriétés.

## Catalogue des contraintes (les plus utilisées)

| Famille      | Contraintes                                                                 | Note                                                               |
|--------------|-----------------------------------------------------------------------------|--------------------------------------------------------------------|
| Base         | `NotBlank`, `NotNull`, `IsNull`, `Blank`, `IsTrue`, `IsFalse`, `Type`       | `NotBlank` refuse `''` et `null` ; `NotNull` accepte `''`          |
| Chaîne       | `Length`, `Regex`, `Email`, `Url`, `Uuid`, `Ulid`, `Ip`, `Hostname`, `Json` | `Email(mode: 'strict')` recommandé                                 |
| Comparaison  | `EqualTo`, `NotEqualTo`, `IdenticalTo`, `GreaterThan`, `LessThan`, `Range`  | `propertyPath: 'startDate'` pour comparer à un autre champ         |
| Nombre       | `Positive`, `PositiveOrZero`, `Negative`, `NegativeOrZero`, `DivisibleBy`   | —                                                                  |
| Date         | `Date`, `DateTime`, `Time`, `Timezone`                                      | Sur string ; sur `\DateTimeInterface` ces contraintes sont inutiles |
| Choix        | `Choice`, `Country`, `Language`, `Locale`, `Currency`                       | `Choice(callback: 'getStatuses')` pour liste dynamique             |
| Fichier      | `File`, `Image`                                                             | `maxSize: '2M'`, `mimeTypes: ['application/pdf']`                  |
| Finance      | `Iban`, `Bic`, `Isbn`, `Issn`, `CardScheme`, `Luhn`                         | —                                                                  |
| Sécurité     | `PasswordStrength`, `NotCompromisedPassword`, `UserPassword`                | `NotCompromisedPassword` appelle l'API HIBP (réseau requis)        |
| Collection   | `Count`, `Unique`, `All`, `Collection`                                      | `All(new Assert\Email())` valide chaque élément                    |
| Objet        | `Valid`, `Cascade`, `Traverse`                                              | `Valid` pour un sous-objet, `Cascade` pour tous les sous-objets    |
| Doctrine     | `UniqueEntity`                                                              | **Attribut de classe**, pas de propriété                           |
| Custom       | `Callback`, `Expression`, `When`, `Compound`, `AtLeastOneOf`, `Sequentially` | Détaillés dans `/symfony:validation-groups`                       |

## Cibles

### Propriété (cas par défaut)

```php
use Symfony\Component\Validator\Constraints as Assert;

class Author
{
    #[Assert\NotBlank(message: 'author.name.blank')]
    #[Assert\Length(min: 2, max: 100)]
    private string $name;

    #[Assert\NotBlank]
    #[Assert\Email(mode: 'strict')]
    private string $email;

    #[Assert\Range(min: 0, max: 120)]
    private int $age;

    #[Assert\Choice(choices: ['draft', 'published', 'archived'])]
    private string $status = 'draft';
}
```

> **Attention propriété typée non initialisée** : `private string $name;` sans valeur par défaut lance un `Error` à la lecture avant affectation. Initialiser (`= ''`) ou rendre nullable si le validator doit gérer le « vide ».

### Getter (valeur calculée)

Nom de méthode commençant par `get`, `is`, `has`. La contrainte porte sur la valeur retournée.

```php
class Author
{
    private string $firstName;
    private string $password;

    #[Assert\IsTrue(message: 'password.not.safe')]
    public function isPasswordSafe(): bool
    {
        return $this->firstName !== $this->password;
    }
}
```

### Classe (invariant multi-propriétés)

#### `UniqueEntity` (Doctrine)

```php
use Doctrine\ORM\Mapping as ORM;
use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;

#[ORM\Entity]
#[UniqueEntity(fields: ['email'], message: 'user.email.already_used')]
#[UniqueEntity(fields: ['tenant', 'slug'], errorPath: 'slug')] // unicité composite
class User
{
    #[ORM\Column]
    private string $email;
}
```

Requiert `symfony/doctrine-bridge`. `errorPath` dirige l'erreur sur un champ form précis.

#### `Callback` de classe

```php
use Symfony\Component\Validator\Context\ExecutionContextInterface;

#[Assert\Callback('validate')]
class Reservation
{
    private \DateTimeImmutable $start;
    private \DateTimeImmutable $end;

    public function validate(ExecutionContextInterface $context): void
    {
        if ($this->end <= $this->start) {
            $context->buildViolation('reservation.end_before_start')
                ->atPath('end')
                ->addViolation();
        }
    }
}
```

Préférer `Expression` pour les règles courtes :

```php
#[Assert\Expression(
    expression: 'this.end > this.start',
    message: 'reservation.end_before_start',
)]
class Reservation { /* ... */ }
```

## Objets imbriqués

`Valid` et `Cascade` ne font **pas** la même chose :

- `#[Assert\Valid]` sur **une propriété** → valide ce sous-objet précis (recommandé, explicite).
- `#[Assert\Cascade]` sur **la classe** → valide récursivement tous les sous-objets typés. Pratique pour un DTO à plusieurs niveaux mais opaque, à utiliser sciemment.

```php
class Order
{
    #[Assert\Valid]
    private Address $billingAddress;

    #[Assert\Valid]
    #[Assert\Count(min: 1, message: 'order.no_lines')]
    private Collection $lines; // chaque OrderLine est validée
}
```

Sans `Valid`, les sous-objets ne sont **pas** parcourus — défaut silencieux fréquent.

## Héritage

- Les contraintes d'une classe mère s'appliquent sur les enfants (fusion, non surcharge).
- Pour surcharger la règle d'un champ parent : retirer la contrainte parent ou la placer dans un groupe (`groups: ['parent']`) et utiliser un groupe différent côté enfant. Détails : `/symfony:validation-groups`.
- Ne pas redéclarer la propriété dans l'enfant juste pour remettre des attributs — PHP le refuse si la propriété mère est `private`, et casse le contrat de substitution si `protected`.

## Validation auto-mappée (Doctrine)

Activée par défaut via `framework.validation.auto_mapping`. Règles déduites :

| Mapping Doctrine                    | Contrainte inférée                 |
|-------------------------------------|------------------------------------|
| `nullable: false`                   | `NotNull`                          |
| `length: N`                         | `Length(max: N)`                   |
| `unique: true` (sur colonne simple) | `UniqueEntity`                     |
| `type: 'integer'` / `'float'`       | `Type`                             |

Pour **désactiver** l'auto-mapping sur une classe ou une propriété :

```php
#[Assert\DisableAutoMapping]
class Legacy { /* ... */ }
```

Ou configurer des exclusions dans `config/packages/validator.yaml` :

```yaml
framework:
    validation:
        auto_mapping:
            'App\Entity\\': []
```

## Déroulement

### 1 — Cadrer

- Classe cible : entité, DTO, value object.
- Règles métier à exprimer.
- Règles déjà couvertes par le mapping Doctrine (ne pas doublonner).
- Groupes ? → si oui, cf. `/symfony:validation-groups`.

### 2 — Poser les contraintes

Ordre typique sur une propriété : présence (`NotBlank`/`NotNull`) → format (`Length`/`Regex`/`Email`) → valeur (`Range`/`Choice`). Les contraintes sont évaluées dans l'ordre de déclaration, toutes exécutées par défaut. Pour court-circuiter au premier échec : `#[Assert\Sequentially([...])]`.

### 3 — Vérifier le mapping

```bash
symfony console debug:validator 'App\Entity\Author'
# ou pour un dossier entier
symfony console debug:validator src/Entity
```

Affiche la liste des contraintes appliquées, leurs groupes, leurs options. À utiliser systématiquement après un changement non trivial.

### 4 — Tester

Les forms appellent le validator automatiquement, les tests functionals sur `/form-handle` couvrent. Pour un test unitaire ciblé :

```php
use Symfony\Component\Validator\Validation;

public function testNameTooShort(): void
{
    $validator = Validation::createValidatorBuilder()
        ->enableAttributeMapping()
        ->getValidator();

    $author = new Author();
    $author->setName('a');

    $violations = $validator->validate($author);

    self::assertCount(1, $violations);
    self::assertSame('name', $violations[0]->getPropertyPath());
}
```

Pour tester une contrainte seule (hors classe) → `/symfony:validation-use`.

### 5 — Clôture

Afficher :

- Classe(s) modifiée(s).
- Contraintes ajoutées / retirées.
- Ce qui reste : traductions dans `translations/validators.*.yaml`, tests, groupes (→ `/symfony:validation-groups`), usage hors form (→ `/symfony:validation-use`).

## Delta Sylius

- Contraintes custom sur une resource Sylius : ajouter `groups: ['Default', 'sylius']`, sinon les forms Sylius (qui valident sur `['Default', 'sylius']`) les ignorent.
- Les validations Sylius vendor sont fournies en XML (`Resources/config/validation.xml` du bundle). Pour les surcharger, `ExtendsValidationFor` ou override de la resource — cf. `/symfony:validation-groups`.

## Argument optionnel

`/symfony:validation-constraints Product` — ajoute/révise les contraintes sur `src/Entity/Product.php`.

`/symfony:validation-constraints src/Dto/CreateOrderRequest.php` — audit d'un DTO (présence, format, auto-mapping désactivé si DTO pur).

`/symfony:validation-constraints` sans argument — demande la classe et les règles à poser.

---
name: object-mapper
description: Mappe un objet PHP vers un autre avec Symfony ObjectMapper (7.3+) — #[Map] (target/source/if/transform), conditions, transformers, MapCollection, multi-cibles. Déclenche sur "mapper un DTO", "DTO vers entité", "entité vers DTO", "#[Map]".
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /object-mapper — Mapper un objet PHP vers un autre avec le composant ObjectMapper

> **Utilise quand** tu transformes une instance PHP en une autre (DTO entrant → entité, entité → DTO de lecture, payload `stdClass` → objet typé) **sans** passer par un format texte intermédiaire (JSON, XML, CSV).
> **Pas quand** tu (dé)sérialises depuis/vers un format texte → `/symfony:serializer-use` couvre tout le pipeline normalize/encode.
> **Pas quand** tu construis une couche API REST complète avec négociation de format → API Platform reste le bon outil.
> **Pas quand** tu fais une simple affectation manuelle de 2-3 propriétés — `new Dto($entity->getId(), $entity->getName())` reste plus lisible que d'invoquer le composant.

Tu déclares **où vont** les valeurs (attributs `#[Map]` côté source ou côté cible) et tu laisses `ObjectMapperInterface::map()` faire le walk : copie des propriétés homonymes, transformations, conditions, récursion détectée. Pas de format texte, pas de parsing — juste de la reflection + property access.

## Détection préalable (obligatoire)

1. Lire `composer.json` — le composant est `symfony/object-mapper`. Sinon `composer require symfony/object-mapper`.
2. Vérifier la version de Symfony : composant introduit en **7.3** (marqué expérimental jusqu'en 7.4). Si le projet est en 7.2 ou moins, refuser l'ajout — proposer plutôt un mapper manuel ou `/symfony:serializer-use` avec un encodage JSON intermédiaire (overkill mais portable).
3. Vérifier `symfony/property-access` — dépendance transitive, mais une `framework.property_access` mal configurée peut faire planter les conditions sur propriétés absentes (cf. plus bas).
4. Vérifier `api-platform/core` — si présent, **ne pas** dupliquer un mapping qui sortirait déjà via une `output` resource Platform. Le mapper a sa place pour le code applicatif interne, pas pour réinventer la couche d'expo API.
5. Si `symfony/serializer` est aussi mobilisé sur le même DTO : clarifier la séparation. Serializer = bord HTTP (JSON ↔ DTO entrant). ObjectMapper = bord domaine (DTO entrant → entité ; entité → DTO sortant). Mélanger les deux sur un même flux est rarement justifié.

## Règles fondamentales

- **ObjectMapper ≠ Serializer**. Le Serializer fait normalize + encode (objet ↔ texte). ObjectMapper fait **objet → objet**, point. Aucun encoder, aucun format. Si la source est un `string` JSON, c'est un job Serializer en amont.
- **Source = lecture, cible = écriture**. Le composant lit les propriétés/`stdClass` de la source via PropertyAccess, écrit dans la cible via PropertyAccess. Donc les invariants Doctrine (via setters), les hooks, les events de propriétés s'appliquent côté cible.
- **Cible : classe ou instance**. `map($source, Target::class)` → instancie via `new Target()` (constructeur **sans args** appelé par défaut). `map($source, $existingTarget)` → mute l'instance fournie (utile pour PATCH partiel sur entité Doctrine).
- **Pas de constructeur à arguments obligatoires côté cible** par défaut. Le composant ne sait pas remplir les args du constructeur — il instancie puis assigne. Si la cible doit être immutable (`readonly` + ctor), passer par une **transformation de classe** (`#[Map(target: …, transform: [Target::class, 'createFrom'])]`) qui fabrique l'objet et retourne l'instance prête.
- **Mapping par homonymie d'abord**. Sans `#[Map(target: …)]`, une propriété source `name` va vers la propriété cible `name` si elle existe. Renommer = ajouter `#[Map(target: 'autreNom')]`. Désactiver = `#[Map(if: false)]`.
- **`#[Map]` côté source ou côté cible — pas les deux** sur la même paire de propriétés. Si c'est déclaré des deux côtés, **la source prime** (cf. doc). Choisir un côté : source quand le DTO entrant connaît sa cible ; cible quand l'entité métier veut rester ignorante du format d'entrée.
- **Conditions : signature stricte**. `ConditionCallableInterface::__invoke(mixed $value, object $source, ?object $target): bool`. Une condition « PHP function » comme `'strlen'` est un raccourci — l'argument c'est `$value`, donc `strlen` sur une string vide retourne `0` (falsy → exclu). Pratique mais piégeux.
- **Transformations : retour utilisé tel quel**. La valeur retournée est assignée à la propriété cible. Type-incompatible côté cible = erreur de PropertyAccess à l'écriture. Pas de coercion magique.
- **Récursion détectée automatiquement** — un graphe avec cycle (User ↔ Manager) est mappé sans boucle infinie : la première occurrence est mémorisée, les suivantes pointent vers l'instance déjà créée.
- **Collections : explicite**. `array` → `array` ne mappe **pas** récursivement les éléments. Il faut `#[Map(transform: new MapCollection())]` sur la propriété pour que chaque item soit re-mappé selon ses propres règles.
- **Multi-cibles ⇒ conditions obligatoires**. Plusieurs `#[Map(target: …)]` au niveau classe sans `if` explicite déclenchent une `MappingException` *Ambiguous mapping*. Une condition par cible, sinon une seule cible.

## Forme minimale : DTO entrant → entité

```php
// src/Dto/CreateProductInput.php
use Symfony\Component\ObjectMapper\Attribute\Map;
use App\Entity\Product;

#[Map(target: Product::class)]
final class CreateProductInput
{
    public function __construct(
        public readonly string $name,
        #[Map(target: 'priceCents', transform: [self::class, 'eurosToCents'])]
        public readonly float $priceEuros,
        #[Map(if: false)]                       // ignoré au mapping
        public readonly ?string $clientToken = null,
    ) {}

    public static function eurosToCents(float $value): int
    {
        return (int) round($value * 100);
    }
}
```

```php
// src/Controller/ProductController.php
use Symfony\Component\ObjectMapper\ObjectMapperInterface;

public function create(
    CreateProductInput $input,
    ObjectMapperInterface $mapper,
    EntityManagerInterface $em,
): JsonResponse {
    /** @var Product $product */
    $product = $mapper->map($input);            // target inféré depuis #[Map(target: …)]
    $em->persist($product);
    $em->flush();

    return new JsonResponse(['id' => $product->getId()], 201);
}
```

`Product` doit avoir un constructeur **sans args obligatoires** (ou `__construct()` vide) pour que `new Product()` fonctionne. Sinon, pattern *factory* via `transform` au niveau classe (cf. plus bas).

## Forme minimale : entité → DTO de lecture (mapping par la cible)

Le DTO de sortie déclare ce qu'il prend depuis l'entité — l'entité reste ignorante :

```php
// src/Dto/ProductView.php
use Symfony\Component\ObjectMapper\Attribute\Map;
use App\Entity\Product;

#[Map(source: Product::class)]
final class ProductView
{
    public string $name = '';

    #[Map(source: 'priceCents', transform: [self::class, 'centsToEuros'])]
    public float $priceEuros = 0.0;

    #[Map(source: 'createdAt')]
    public ?\DateTimeImmutable $createdAt = null;

    public static function centsToEuros(int $value): float
    {
        return $value / 100;
    }
}

// Usage
$view = $mapper->map($product, ProductView::class);
```

Pattern recommandé : un `ProductView` par cas d'usage (`ProductCardView`, `ProductDetailView`, `ProductAdminView`) plutôt qu'un DTO unique avec quinze groupes.

## Attribut `#[Map]` — paramètres

| Paramètre | Type | Usage |
|-----------|------|-------|
| `target` | `string\|class-string` | Au niveau classe : classe cible. Au niveau propriété : nom de la propriété cible. |
| `source` | `string\|class-string` | Au niveau classe : classe source attendue. Au niveau propriété : nom de la propriété source à lire. |
| `if` | `bool\|callable\|class-string` | Condition d'inclusion. `false` désactive. Service `ConditionCallableInterface` pour logique riche. |
| `transform` | `callable\|class-string` | Transforme la valeur lue avant écriture. Service `TransformCallableInterface` pour logique avec accès à `$source` / `$target`. |

`#[Map]` est répétable au niveau classe (pour multi-cibles avec conditions) **et** au niveau propriété (pour cibler plusieurs propriétés différentes selon la classe cible).

## Conditional mapping

### Condition simple (callable PHP)

```php
#[Map(if: 'strlen')]                            // vrai si valeur non vide string
public ?string $discountCode = null;

#[Map(if: [Order::class, 'isShippable'])]       // méthode statique
public ?string $shippingAddress = null;
```

### Condition via service (`ConditionCallableInterface`)

```php
use Symfony\Component\ObjectMapper\ConditionCallableInterface;

final class IsAdultCondition implements ConditionCallableInterface
{
    public function __invoke(mixed $value, object $source, ?object $target): bool
    {
        return $value instanceof \DateTimeInterface
            && $value->diff(new \DateTimeImmutable())->y >= 18;
    }
}

#[Map(if: IsAdultCondition::class)]
public \DateTimeImmutable $birthDate;
```

Autoconfigure le tag (Symfony résout via le `conditionCallableLocator`). Pas besoin de déclarer le service manuellement si autowiring + autoconfigure sont actifs.

### Condition par cible (`TargetClass`)

Quand la même classe source mappe vers plusieurs cibles, conditionner par classe cible :

```php
use Symfony\Component\ObjectMapper\Condition\TargetClass;

#[Map(target: AdminUserView::class)]
#[Map(target: PublicUserView::class)]
final class User
{
    public string $name = '';

    #[Map(target: 'lastLoginIp', if: new TargetClass(AdminUserView::class))]
    public ?string $lastLoginIp = null;          // visible uniquement côté admin
}
```

### Configuration PropertyAccess pour conditions sur propriétés absentes

Si une condition lit une propriété qui peut être manquante (cas legacy / payload partiel), désactiver l'exception stricte sur PropertyAccess :

```yaml
# config/packages/framework.yaml
framework:
    property_access:
        throw_exception_on_invalid_property_path: false
```

Sinon une `NoSuchPropertyException` interrompt le mapping global.

## Transformations

### Callable PHP / méthode statique / closure

```php
#[Map(transform: 'intval')]                                      // fonction PHP
#[Map(transform: [PriceFormatter::class, 'format'])]             // méthode statique
#[Map(transform: fn (float $v): string => number_format($v, 2))] // closure
```

Signature acceptée : `function (mixed $value, object $source = ?, ?object $target = ?): mixed`. Les arguments suivants sont passés mais non requis — `'intval'` (qui prend 1 arg) fonctionne, PHP ignore les surplus pour les built-ins.

### Service `TransformCallableInterface`

Pour une transformation qui dépend d'autres services ou de l'objet source complet :

```php
use Symfony\Component\ObjectMapper\TransformCallableInterface;

final class FullNameTransformer implements TransformCallableInterface
{
    public function __invoke(mixed $value, object $source, ?object $target): mixed
    {
        return trim($source->firstName.' '.$source->lastName);
    }
}

#[Map(target: 'fullName', transform: FullNameTransformer::class)]
public string $firstName = '';                  // $value = $source->firstName
```

### Transformation de classe (factory pour cible immutable)

Quand la cible n'a pas de constructeur sans args — typiquement DTO `final` `readonly` :

```php
#[Map(target: Product::class, transform: [Product::class, 'createFromInput'])]
final class CreateProductInput
{
    public function __construct(
        public readonly string $name,
        public readonly int $priceCents,
    ) {}
}

final class Product
{
    private function __construct(
        public readonly string $name,
        public readonly int $priceCents,
        public readonly \DateTimeImmutable $createdAt,
    ) {}

    public static function createFromInput(mixed $value, object $source): self
    {
        return new self($source->name, $source->priceCents, new \DateTimeImmutable());
    }
}
```

La transformation de classe **remplace** l'instanciation par défaut. Les `#[Map]` au niveau propriétés sont ensuite appliqués sur l'objet retourné — donc attention aux propriétés `readonly` : elles sont déjà figées par le constructeur, les `#[Map]` propriétés ne pourront pas les écraser (PropertyAccess lèvera).

### Mapper une collection : `MapCollection`

Sans `MapCollection`, un `array` source est copié tel quel — les éléments ne sont pas re-mappés.

```php
use Symfony\Component\ObjectMapper\Transform\MapCollection;

final class OrderInput
{
    /** @var ProductInput[] */
    #[Map(transform: new MapCollection())]
    public array $items = [];
}
```

Chaque `ProductInput` du tableau est mappé selon ses propres règles `#[Map]`. Le résultat reste un `array` côté cible (pas une `Collection` Doctrine — pour ça, `transform` custom qui wrappe).

## Mapping multi-cibles

La même classe source peut produire plusieurs cibles, conditionnées par l'état :

```php
#[Map(target: OnlineEvent::class,   if: [self::class, 'isOnline'])]
#[Map(target: PhysicalEvent::class, if: [self::class, 'isPhysical'])]
final class EventInput
{
    public function __construct(
        public readonly string $type,
        public readonly string $title,
    ) {}

    public static function isOnline(mixed $value, object $source): bool
    {
        return $source->type === 'online';
    }

    public static function isPhysical(mixed $value, object $source): bool
    {
        return $source->type === 'physical';
    }
}

$event = $mapper->map(new EventInput('physical', 'PHP Forum'));   // → PhysicalEvent
```

Conditions **obligatoires** sur chaque `#[Map(target: …)]` au niveau classe sinon `Ambiguous mapping`. Si on `map($input, OnlineEvent::class)` explicitement, les conditions sont court-circuitées — l'appelant a déjà tranché.

## Récursion (cycles dans le graphe)

Le composant détecte les cycles et réutilise les instances déjà mappées. Aucun setup particulier requis :

```php
#[Map(target: UserDto::class)]
final class User
{
    public string $name = '';
    public ?User $manager = null;
}

#[Map(source: User::class)]
final class UserDto
{
    public string $name = '';
    public ?UserDto $manager = null;
}

$bob = new User(); $bob->name = 'Bob';
$alice = new User(); $alice->name = 'Alice';
$bob->manager = $alice;
$alice->manager = $bob;                          // cycle

$bobDto = $mapper->map($bob, UserDto::class);    // pas d'infinite loop
```

À comparer avec le Serializer qui exige un `circular_reference_handler` pour la même situation.

## Décorer `ObjectMapperInterface`

Cas d'usage : logging, métriques, contexte multi-tenant, cache des graphes mappés.

```php
use Symfony\Component\DependencyInjection\Attribute\AsDecorator;
use Symfony\Component\ObjectMapper\ObjectMapperAwareInterface;
use Symfony\Component\ObjectMapper\ObjectMapperInterface;

#[AsDecorator(decorates: ObjectMapperInterface::class)]
final class LoggingObjectMapper implements ObjectMapperInterface
{
    private readonly ObjectMapperInterface $decorated;

    public function __construct(
        ObjectMapperInterface $decorated,
        private readonly LoggerInterface $logger,
    ) {
        $this->decorated = $decorated instanceof ObjectMapperAwareInterface
            ? $decorated->withObjectMapper($this)
            : $decorated;
    }

    public function map(object $source, object|string|null $target = null): object
    {
        $this->logger->debug('Mapping', [
            'source' => $source::class,
            'target' => is_string($target) ? $target : ($target ? $target::class : 'inferred'),
        ]);
        return $this->decorated->map($source, $target);
    }
}
```

Une propriété `readonly` ne pouvant être assignée qu'une fois, on reçoit `$decorated` en argument non promu et on résout la version finale (avec ou sans `withObjectMapper`) dans une seule affectation.

Le `withObjectMapper($this)` est important : il garantit que les sous-mappings récursifs passent **aussi** par le décorateur, pas par le mapper original.

## Mapper applicatif (style MapStruct)

Quand on veut concentrer toutes les règles de mapping dans une **classe mapper dédiée** plutôt que sur les DTO/entités (séparation source/règles) :

```php
use Symfony\Component\ObjectMapper\Attribute\Map;
use Symfony\Component\ObjectMapper\Metadata\MapStructMapperMetadataFactory;
use Symfony\Component\ObjectMapper\ObjectMapper;
use Symfony\Component\ObjectMapper\ObjectMapperInterface;

#[Map(source: LegacyUser::class, target: UserDto::class)]
final class LegacyUserMapper implements ObjectMapperInterface
{
    private readonly ObjectMapperInterface $objectMapper;

    public function __construct(?ObjectMapperInterface $objectMapper = null)
    {
        $factory = new MapStructMapperMetadataFactory(self::class);
        $this->objectMapper = $objectMapper ?? new ObjectMapper($factory);
    }

    #[Map(source: 'fullName', target: 'name')]
    #[Map(source: 'emailAddr', target: 'email')]
    public function map(object $source, object|string|null $target = null): object
    {
        return $this->objectMapper->map($source, $target);
    }
}
```

Avantage : les classes métier restent vierges d'attributs `#[Map]`. Inconvénient : un mapper par paire source/cible, plus de plomberie. À réserver aux projets où DTO et entités sont dans des bounded contexts différents et ne doivent pas se connaître.

## Patch partiel (édition d'une entité)

Pour appliquer un DTO entrant sur une entité Doctrine **existante** sans la réinstancier :

```php
$product = $repository->find($id) ?? throw new NotFoundHttpException();

$mapper->map($input, $product);                  // mute l'instance fournie
$em->flush();
```

Toutes les propriétés du DTO sont écrites sur `$product`. **Limite** : pas de notion de « champ absent du payload » — un `null` côté DTO écrira `null` côté entité. Pour un vrai PATCH (seulement les champs présents), utiliser `#[Map(if: …)]` avec une condition sur valeur non-null **ou** dénormaliser via Serializer + `OBJECT_TO_POPULATE` (le Serializer respecte mieux la sémantique « champs présents uniquement »).

## ObjectMapper vs Serializer — quand choisir quoi

| Critère | ObjectMapper | Serializer |
|---------|--------------|------------|
| Source / cible | Objet PHP ↔ Objet PHP | Objet PHP ↔ texte (json/xml/csv/yaml) |
| Bord HTTP | Non | Oui |
| Performances | Reflection + PropertyAccess uniquement | Pipeline normalize + encode |
| API | `map($source, $target)` | `serialize`, `deserialize`, `normalize`, `denormalize` |
| Récursion | Auto (cycles détectés) | Manuel (`circular_reference_handler`) |
| Groupes | Non (utiliser conditions ou DTO dédiés) | `#[Groups]` |
| Mappings multiples cibles | `#[Map]` répétable + `if` | DTO distincts |
| Polymorphisme | `TargetClass` + conditions | `#[DiscriminatorMap]` |
| Composer | `symfony/object-mapper` | `symfony/serializer` |

**Heuristique** : pipeline HTTP entrant → Serializer dénormalise vers DTO (couche bord). DTO → entité interne → ObjectMapper. Entité → DTO sortant → ObjectMapper. DTO sortant → réponse HTTP → Serializer normalise + encode.

## Pièges fréquents

- **Constructeur cible avec args obligatoires** : `new Product()` plante. Solutions : ajouter une `transform` de classe (factory statique), ou rendre les args du ctor optionnels.
- **`array` non re-mappé** : on attend que les enfants d'une collection soient mappés, ils sont copiés tels quels. Toujours `#[Map(transform: new MapCollection())]` sur les propriétés `array` typées.
- **`Ambiguous mapping`** : plusieurs `#[Map(target: …)]` au niveau classe sans `if`. Ajouter une condition par cible.
- **Condition `'strlen'` qui se comporte étrangement** : `strlen('')` retourne `0` (falsy), donc une string vide est exclue. Pour inclure les strings vides, condition explicite.
- **`#[Map]` source ET cible sur la même paire** : la source prime, l'autre est ignorée silencieusement. Choisir un côté et y rester pour cette propriété.
- **Propriété `readonly` côté cible avec `#[Map]` propriété** : PropertyAccess lève à l'écriture. `readonly` exige une factory de classe (ctor unique).
- **`NoSuchPropertyException` sur condition d'une propriété optionnelle** : activer `framework.property_access.throw_exception_on_invalid_property_path: false`.
- **Mapping silencieux d'une propriété privée non accessible** : sans setter ni propriété publique, PropertyAccess n'écrit rien — pas d'erreur, juste un champ vide. Vérifier l'accessibilité via getter/setter ou propriété publique.
- **Confusion avec le Serializer** : on tente de `map()` un payload JSON brut (`string`). Le composant attend un **objet** (`stdClass` ou typé) — passer par `json_decode($json)` avant pour obtenir un `stdClass`.
- **Décorateur sans `withObjectMapper($this)`** : les sous-mappings récursifs passent par le mapper original, le décorateur n'est pas traversé. Casse les métriques / le logging au-delà du premier niveau.
- **Doctrine + `map()` direct sur entité managée** : les listeners `prePersist` / `preUpdate` ne savent pas distinguer un mapping d'un changement métier. Si l'entité est managée et qu'on mute via `map()`, le `flush()` qui suit déclenche les hooks comme un update normal — comportement attendu, mais à garder en tête (ne pas mapper « pour voir »).

## Déroulement

### 1 — Cadrer

- Source et cible : deux **objets** PHP ? (Sinon → Serializer.)
- Direction : DTO → entité (création), DTO → entité existante (PATCH), entité → DTO de lecture, entité → entité (clonage typé).
- Cible : nouvelle instance ou mutation d'une instance existante (`map($src, $existing)`) ?
- Constructeur cible : sans args ou avec args obligatoires (→ factory via `transform`) ?
- Transformations nécessaires : conversions d'unité (cents/euros, secondes/ISO), concaténations (firstName+lastName), normalisations (uppercase, trim).
- Conditions : champs admin-only (`TargetClass`), champs ignorés (`if: false`), inclusion conditionnelle.
- Cas multi-cibles : une seule source pour plusieurs cibles selon état (`if` par cible).
- Collections présentes : prévoir `MapCollection` sur chaque propriété `array` typée.

### 2 — Implémenter

- `composer require symfony/object-mapper` si absent.
- Choisir la stratégie d'attribut : côté source (DTO entrant connaît la cible) **ou** côté cible (DTO sortant lit l'entité). Pas les deux.
- DTO + `#[Map(target|source: …)]` au niveau classe ; `#[Map]` au niveau propriété pour renommer / transformer / conditionner.
- Si cible immutable : factory statique via `#[Map(target: …, transform: [Target::class, 'createFrom'])]`.
- Si conditions riches : implémenter `ConditionCallableInterface`.
- Si transformations contextuelles : implémenter `TransformCallableInterface`.
- Côté appelant : injecter `ObjectMapperInterface`, appeler `map($source, $target)`.
- Tests : un test unitaire par paire source/cible, vérifier les mappings nominaux, les conditions actives/inactives, les cycles si applicables.

### 3 — Vérifier

```bash
composer show symfony/object-mapper                 # version installée
symfony console debug:autowiring ObjectMapperInterface
symfony console debug:container --tag=object_mapper.transform_callable
symfony console debug:container --tag=object_mapper.condition_callable
vendor/bin/phpstan analyse src
```

Test : un PHPUnit qui appelle `$mapper->map(new SourceFixture(), Target::class)` et asserte les valeurs cibles (préférer `assertEquals` sur des graphes d'objets, `assertSame` sur des scalaires).

### 4 — Clôture

Afficher :

- Direction du mapping (source → cible, ou paires multiples).
- Côté qui porte les `#[Map]` (source ou cible) et raison du choix.
- Transformations / conditions / multi-cibles déclarées.
- Collections gérées via `MapCollection` ?
- Stratégie pour cible immutable (factory) si applicable.
- Ce qui reste : couvrir les cas limites (null, collections vides) en tests, documenter les paires source/cible (un README court par bounded context aide).

## Delta Sylius

- Sylius v1 utilise massivement des **factories** (`ProductFactoryInterface`, `OrderFactoryInterface`) pour construire ses resources avec leurs invariants (channels, locales, taxons par défaut). **Ne pas** remplacer une factory Sylius par un `map()` ObjectMapper — la factory porte de la logique métier (génération de slug, association de variantes par défaut, etc.) qu'un mapping perd.
- Pour un **DTO custom au-dessus d'une resource Sylius** (export, projection, payload B2B), ObjectMapper est en revanche bien placé : pas besoin de toucher la resource vendor, le DTO déclare ce qu'il prend via `#[Map(source: Product::class)]`.
- Sylius v2 commence à exposer des DTOs plus modernes (resource v2 / API Platform). Si le projet est sur Sylius v2, vérifier qu'un mapping ne duplique pas un mapping déjà fait par Platform — la `Pipe` de Platform peut suffire.
- Les **Translations Sylius** (`ProductTranslation` par locale) ne sont **pas** mappables naïvement : la propriété `translations` est une `Collection`, et il faut résoudre la locale courante. Soit transformation custom qui prend `LocaleContextInterface`, soit DTO qui n'expose qu'une chaîne `name` après résolution côté source.
- Sur les **channels** : un `Product` Sylius est rattaché à plusieurs channels ; le DTO sortant doit savoir lequel exposer. Injecter `ChannelContextInterface` dans un `TransformCallableInterface` plutôt que de propager le contexte channel à travers tout le code applicatif.

## Argument optionnel

`/symfony:object-mapper CreateProductInput` — scaffolde un DTO entrant `final` avec `#[Map(target: Product::class)]`, propriétés `readonly`, exemples de `#[Map(target: …, transform: …)]` sur les conversions usuelles. Lit `Product` existant pour caler les noms de propriétés.

`/symfony:object-mapper "Product → ProductView"` — scaffolde le DTO de lecture côté cible avec `#[Map(source: Product::class)]`, transformations centisme→euros et formatage des dates.

`/symfony:object-mapper FullNameTransformer` — scaffolde un `TransformCallableInterface` avec signature complète, autoconfigure laisse Symfony tagger.

`/symfony:object-mapper IsAdultCondition` — scaffolde une `ConditionCallableInterface`.

`/symfony:object-mapper "multi-target EventInput"` — scaffolde une source avec deux `#[Map(target: …, if: …)]` au niveau classe et les méthodes statiques de discrimination.

`/symfony:object-mapper src/Dto/CreateProductInput.php` — audit d'un DTO existant : `#[Map]` cohérents (pas de mix source/cible ambigu), conditions sur les optionnels, `MapCollection` présent sur les `array` typés, factory si cible `readonly`, pas de propriété `readonly` ciblée par `#[Map]` propriété.

`/symfony:object-mapper` sans argument — demande le cas (DTO entrant, DTO sortant, transformer custom, condition custom, multi-cibles, audit, migration depuis un mapper manuel).

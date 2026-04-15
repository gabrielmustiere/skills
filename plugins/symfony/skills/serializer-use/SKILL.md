---
name: serializer-use
description: Conçoit la (dé)sérialisation Symfony — SerializerInterface, normalizers/encoders (json/xml/csv), attributs #[Groups] / #[Context] / #[DiscriminatorMap]. Déclenche sur "sérialiser Symfony", "désérialiser JSON", "#[Groups]", "JSON vers DTO".
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /serializer-use — (Dé)sérialiser avec le composant Symfony Serializer

> **Utilise quand** tu exposes un objet PHP dans une réponse HTTP (JSON/XML), tu hydrates un DTO depuis un payload entrant, tu exportes des entités en CSV, ou tu partages des payloads structurés entre services.
> **Pas quand** tu construis une API REST complète avec pagination, filtres, hypermedia → API Platform couvre tout ça et s'appuie déjà sur le Serializer en sous-main.
> **Pas quand** tu mappes des DTO entre formes PHP équivalentes sans passer par un format texte → `symfony/object-mapper` (Symfony 7.3+) est plus direct.
> **Pas quand** tu veux juste `json_encode` un tableau associatif trivial — inutile d'invoquer le composant pour ça.

Tu sépares **normalisation** (objet ↔ tableau) et **encodage** (tableau ↔ format texte). Le Serializer enchaîne les deux via un pipeline de normalizers (un par type géré) et d'encoders (un par format). Les attributs PHP portent la configuration au plus près des classes — pas de config YAML géante.

## Détection préalable (obligatoire)

1. Lire `composer.json` — vérifier `symfony/serializer`. Sinon `composer require symfony/serializer`.
2. Vérifier `symfony/serializer-pack` si on veut les extras (`property-access`, `property-info`, `doctrine/annotations` si anciens tags, `phpdocumentor/reflection-docblock` pour la déduction de types via PHPDoc). Sans `property-info` + reflection-docblock, la dénormalisation d'une collection typée via `@var Item[]` ne marche pas.
3. Vérifier `config/packages/serializer.yaml` / `framework.yaml` — l'annotation `enable_attributes: true` est par défaut en Symfony 6.1+, mais `name_converter` et `circular_reference_handler` se configurent là.
4. Vérifier `symfony/uid` si des `Ulid`/`Uuid` circulent — le `UidNormalizer` est auto-wire mais nécessite le paquet.
5. Si `api-platform/core` est installé → **ne pas** dupliquer des `#[Groups]` que Platform déclare déjà dans ses resources ; aligner la nomenclature (`product:read`, `product:write`).

## Règles fondamentales

- **Serializer = Normalizer + Encoder**. `serialize($obj, 'json', $ctx)` = normalize → encode. `deserialize($payload, Class::class, 'json', $ctx)` = decode → denormalize. Ne jamais appeler `json_encode` sur un résultat de `normalize()` — passer par `serialize()` garantit un encoder cohérent.
- **Type cible explicite à la dénormalisation** : `deserialize($json, Order::class, 'json')`. Pour une collection, `Order[]::class` n'existe pas — utiliser `Order::class.'[]'` (la magie string) ou `AbstractNormalizer::OBJECT_TO_POPULATE` + une collection pré-instanciée.
- **Ne jamais exposer une entité Doctrine directement** en sortie d'API publique. Les lazy proxies, les relations bidirectionnelles, les mots-de-passe hashés et les champs internes (`isDeleted`, `internalNotes`) fuitent. Toujours passer par un DTO de lecture ou par `#[Groups]` strictement appliqués.
- **`#[Groups]` ou rien** dès qu'un objet a plus de 3-4 propriétés. Sans groupes, la sérialisation expose tout ce qui a un getter public. Convention : `entité:read`, `entité:write`, `entité:read:admin`.
- **Immutabilité côté DTO entrant** : dénormaliser vers une classe `final` avec propriétés `readonly` et un constructeur qui reçoit le payload. `new Order(orderId: 42, items: [...])` est mieux qu'un objet à setters — les invariants sont vérifiés à la construction.
- **Pas de services dans les DTO** : un DTO est un sac de données. La logique métier reste dans des services qui consomment le DTO.
- **Circular reference** : une relation bidirectionnelle (Order ↔ Item) fait boucler le normalizer par défaut. Ajouter un `circular_reference_handler` global (retourne l'`id`) ou scope les groupes pour couper la branche retour.
- **Format stable ≠ PHP stable** : ne pas exposer `camelCase` PHP en `camelCase` JSON par accident si la convention publique est `snake_case`. Utiliser un `NameConverter` (cf. `CamelCaseToSnakeCaseNameConverter`) appliqué globalement.
- **Erreurs de dénormalisation = 400, pas 500** : attraper `NotNormalizableValueException` / `PartialDenormalizationException` dans les contrôleurs/API et renvoyer 400 avec les erreurs de champ. Sinon Symfony renvoie une 500 générique.
- **Contexte = configuration, pas état** : tout ce qui change selon l'appel (groups, skip_null_values, datetime_format) passe par le 3e argument `$context`. Ne pas subclasser un normalizer juste pour changer une option.

## Forme minimale : sérialiser un objet en JSON

```php
use Symfony\Component\Serializer\SerializerInterface;
use Symfony\Component\Serializer\Attribute\Groups;

final class Product
{
    public function __construct(
        #[Groups(['product:read'])]
        public readonly int $id,
        #[Groups(['product:read', 'product:write'])]
        public readonly string $name,
        #[Groups(['product:read'])]
        public readonly int $priceCents,
    ) {}
}

final class ProductController
{
    public function __construct(private readonly SerializerInterface $serializer) {}

    public function show(Product $product): JsonResponse
    {
        $json = $this->serializer->serialize(
            $product,
            'json',
            ['groups' => ['product:read']],
        );

        return new JsonResponse($json, 200, [], json: true); // $json déjà encodé
    }
}
```

Le `json: true` du `JsonResponse` évite un double-encodage.

## Forme minimale : dénormaliser un payload entrant

```php
use Symfony\Component\Serializer\Exception\NotNormalizableValueException;
use Symfony\Component\Serializer\Exception\PartialDenormalizationException;
use Symfony\Component\Serializer\Normalizer\AbstractNormalizer;

final class CreateProductInput
{
    public function __construct(
        #[Assert\NotBlank]
        public readonly string $name,
        #[Assert\Positive]
        public readonly int $priceCents,
    ) {}
}

public function create(Request $request): JsonResponse
{
    try {
        /** @var CreateProductInput $input */
        $input = $this->serializer->deserialize(
            $request->getContent(),
            CreateProductInput::class,
            'json',
            [AbstractNormalizer::DISABLE_TYPE_ENFORCEMENT => false],
        );
    } catch (PartialDenormalizationException $e) {
        return $this->errorResponse($e->getErrors());
    }

    $violations = $this->validator->validate($input);
    if (count($violations) > 0) {
        return $this->violationsResponse($violations);
    }

    // ... créer le produit à partir du DTO ...
}
```

La **dénormalisation typée** (constructeur avec types stricts) attrape les payloads malformés avant même la validation — mais ne remplace pas le Validator (règles métier, format d'email, etc.).

`COLLECT_DENORMALIZATION_ERRORS => true` dans le contexte fait remonter **toutes** les erreurs de type dans une `PartialDenormalizationException` au lieu de s'arrêter à la première.

## Normalizers livrés

Autowire par ordre de priorité (le premier qui `supportsNormalization($data)` gagne) :

- **`DateTimeNormalizer`** — `\DateTimeInterface`. Option `datetime_format` dans le contexte (`DateTimeInterface::ATOM` par défaut). Attention : dénormalise vers `\DateTime` par défaut, pas `DateTimeImmutable` — forcer via `'datetime_format' => …` + hint de type `DateTimeImmutable` dans le constructeur.
- **`UidNormalizer`** — `Symfony\Component\Uid\Uuid` / `Ulid`. Format par défaut `canonical` (string), options `base58` / `base32` / `rfc4122`.
- **`BackedEnumNormalizer`** — enums PHP 8.1 backed. Non-backed enums = non-sérialisable (par design).
- **`DateIntervalNormalizer`** — `\DateInterval` en `PnYnMnDTnHnMnS` ISO 8601.
- **`JsonSerializableNormalizer`** — bascule sur la méthode `jsonSerialize()` si l'objet l'implémente. Simple mais couple format JSON à la classe — à éviter quand plusieurs formats sont prévus.
- **`ObjectNormalizer`** — défaut générique. Utilise le PropertyAccess : getters publics, propriétés publiques, PropertyInfo pour typer. Le plus puissant, aussi le plus lent — activer `NormalizerInterface::MAX_DEPTH_HANDLER` et `ATTRIBUTES` pour couper le graphe.
- **`GetSetMethodNormalizer`** — ne regarde que les getters/setters. Plus rapide, utile quand les propriétés n'ont pas d'accès public direct.
- **`PropertyNormalizer`** — accès direct aux propriétés (même privées via reflection). Contourne les getters — à réserver aux DTO plats.
- **`ArrayDenormalizer`** — gère `Type::class.'[]'` lors de la dénormalisation d'une collection.

Priorité : place un normalizer custom avec `priority` > ObjectNormalizer (typiquement `64`) pour qu'il soit essayé avant.

## Encoders livrés

- `JsonEncoder` — format `json`. Options : `json_encode_options` (ex: `JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT`), `json_decode_associative`.
- `XmlEncoder` — format `xml`. Options : `xml_root_node_name`, `xml_format_output`, `remove_empty_tags`.
- `CsvEncoder` — format `csv`. Options : `csv_delimiter`, `csv_enclosure`, `csv_escape_char`, `csv_headers` (forcer l'ordre), `as_collection` (lecture de plusieurs lignes).
- `YamlEncoder` — format `yaml`. Nécessite `symfony/yaml`.

Un seul encoder gère l'encode **et** le decode pour son format.

## Attributs essentiels

### `#[Groups]`

```php
use Symfony\Component\Serializer\Attribute\Groups;

final class Order
{
    #[Groups(['order:read'])]
    public int $id;

    #[Groups(['order:read', 'order:write'])]
    public string $reference;

    #[Groups(['order:read:admin'])]
    public ?string $internalNote;
}

$this->serializer->serialize($order, 'json', ['groups' => ['order:read']]);
```

Un champ sans groupe déclaré **n'apparaît pas** si `groups` est fourni dans le contexte. Sans `groups` dans le contexte, tous les champs passent (mode permissif — rarement voulu en prod).

### `#[SerializedName]`

Renommer un champ sans toucher au PHP :

```php
#[SerializedName('product_reference')]
public readonly string $ref;
```

Utile pour respecter une convention externe sans renommer la propriété.

### `#[SerializedPath]`

Aplatir / reconstruire un chemin imbriqué :

```php
final class AddressDto
{
    public function __construct(
        #[SerializedPath('[address][city]')]
        public readonly string $city,
    ) {}
}
// Entrée : {"address": {"city": "Lyon"}} → $dto->city === 'Lyon'
```

Évite de créer un sous-DTO juste pour traverser un niveau.

### `#[Ignore]`

Retire un champ quel que soit le groupe. Plus robuste qu'`AbstractNormalizer::IGNORED_ATTRIBUTES` qui met la logique côté appel.

### `#[Context]`

Fige un contexte par propriété — typique pour un format de date spécifique à un champ :

```php
#[Context([DateTimeNormalizer::FORMAT_KEY => 'Y-m-d'])]
public readonly \DateTimeImmutable $deliveryDate;
```

On peut aussi scoper par direction (`normalizationContext` / `denormalizationContext`) et par groupe.

### `#[MaxDepth]`

Limite la profondeur de récursion. Nécessite `AbstractObjectNormalizer::ENABLE_MAX_DEPTH => true` dans le contexte.

```php
final class Category
{
    #[MaxDepth(2)]
    public Collection $children;
}
```

### `#[DiscriminatorMap]`

Dénormalisation polymorphique :

```php
#[DiscriminatorMap(typeProperty: 'type', mapping: [
    'card'     => CardPayment::class,
    'transfer' => BankTransferPayment::class,
])]
abstract class Payment { /* ... */ }
```

Le JSON d'entrée doit porter `"type": "card"` — le Serializer choisit la classe concrète.

## Groupes par direction (read / write)

Le pattern `entité:read` + `entité:write` suffit tant que les deux schémas sont proches. Dès qu'ils divergent beaucoup, préférer **deux DTO distincts** (`ProductView` et `CreateProductInput`) — moins de groupes imbriqués, contrats plus lisibles.

## NameConverter

Pour convertir globalement `camelCase` PHP ↔ `snake_case` JSON :

```yaml
# config/packages/serializer.yaml
framework:
    serializer:
        name_converter: 'serializer.name_converter.camel_case_to_snake_case'
```

Combiner avec `#[SerializedName]` par champ grâce à `MetadataAwareNameConverter` : le `#[SerializedName]` a priorité, le reste passe par la conversion globale.

## Circular reference handler

```php
use Symfony\Component\Serializer\Normalizer\AbstractObjectNormalizer;

$this->serializer->serialize($order, 'json', [
    AbstractObjectNormalizer::CIRCULAR_REFERENCE_HANDLER => fn ($object) => $object->getId(),
]);
```

Ou globalement dans `serializer.yaml` :

```yaml
framework:
    serializer:
        circular_reference_handler: App\Serializer\CircularReferenceHandler
```

Le handler reçoit l'objet qui boucle, retourne sa représentation courte (id, iri). **Alternative préférable** : couper la branche retour avec `#[Groups]` ou `#[Ignore]`, on évite l'ambiguïté.

## Normalizer custom

Pour typer un objet qui ne rentre dans aucun normalizer livré :

```php
use Symfony\Component\Serializer\Normalizer\NormalizerInterface;
use Symfony\Component\Serializer\Normalizer\DenormalizerInterface;

final class MoneyNormalizer implements NormalizerInterface, DenormalizerInterface
{
    public function normalize(mixed $data, ?string $format = null, array $context = []): array
    {
        /** @var Money $data */
        return ['amount' => $data->amount, 'currency' => $data->currency];
    }

    public function supportsNormalization(mixed $data, ?string $format = null, array $context = []): bool
    {
        return $data instanceof Money;
    }

    public function denormalize(mixed $data, string $type, ?string $format = null, array $context = []): Money
    {
        return new Money($data['amount'], $data['currency']);
    }

    public function supportsDenormalization(mixed $data, string $type, ?string $format = null, array $context = []): bool
    {
        return $type === Money::class;
    }

    public function getSupportedTypes(?string $format): array
    {
        return [Money::class => true]; // true = cacheable
    }
}
```

Autoconfigure ajoute le tag `serializer.normalizer`. `getSupportedTypes()` (Symfony 6.3+) est **obligatoire** pour le cache — sans, le normalizer reste utilisable mais bypass le cache, pénalisant les perfs sur les gros graphes.

Pour prioriser :

```yaml
# config/services.yaml
services:
    App\Serializer\MoneyNormalizer:
        tags:
            - { name: 'serializer.normalizer', priority: 100 }
```

## Contexte utile

| Option                                                        | Usage                                                                                                  |
|---------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| `groups`                                                      | Filtre par `#[Groups]`                                                                                 |
| `AbstractNormalizer::ATTRIBUTES`                              | Liste blanche d'attributs (prime sur groups)                                                           |
| `AbstractNormalizer::IGNORED_ATTRIBUTES`                      | Liste noire d'attributs                                                                                |
| `AbstractNormalizer::OBJECT_TO_POPULATE`                      | Dénormalise sur un objet existant (édition partielle PATCH)                                            |
| `AbstractNormalizer::ALLOW_EXTRA_ATTRIBUTES => false`         | Refuse les champs inconnus en dénormalisation (préférer en API pour détecter les payloads erronés)     |
| `AbstractNormalizer::DEFAULT_CONSTRUCTOR_ARGUMENTS`           | Valeurs par défaut pour les args du constructeur si absents du payload                                 |
| `AbstractObjectNormalizer::SKIP_NULL_VALUES => true`          | Omet les champs `null` en sortie (évite un JSON pollué)                                                |
| `AbstractObjectNormalizer::SKIP_UNINITIALIZED_VALUES => true` | Ignore les propriétés jamais initialisées (évite l'erreur de reflection)                               |
| `AbstractObjectNormalizer::PRESERVE_EMPTY_OBJECTS => true`    | Préserve un `{}` au lieu de `[]` pour un objet vide (gros piège JSON)                                  |
| `AbstractObjectNormalizer::MAX_DEPTH_HANDLER`                 | Callback appelé quand `#[MaxDepth]` est atteint                                                        |
| `DateTimeNormalizer::FORMAT_KEY`                              | Format DateTime (`Y-m-d`, `DateTimeInterface::ATOM`, etc.)                                             |
| `DateTimeNormalizer::TIMEZONE_KEY`                            | Timezone de dénormalisation                                                                            |
| `JsonEncoder::OPTIONS`                                        | Flags `json_encode` (`JSON_UNESCAPED_UNICODE`, `JSON_PRETTY_PRINT`)                                    |
| `CsvEncoder::HEADERS_KEY`                                     | Ordre des colonnes CSV                                                                                 |
| `AbstractObjectNormalizer::COLLECT_DENORMALIZATION_ERRORS`    | Remonte toutes les erreurs de type d'un coup via `PartialDenormalizationException`                     |

## Patch partiel (édition)

Un PATCH sur une ressource existante se fait via `OBJECT_TO_POPULATE` :

```php
$product = $repository->find($id) ?? throw new NotFoundHttpException();

$this->serializer->deserialize(
    $request->getContent(),
    Product::class,
    'json',
    [
        AbstractNormalizer::OBJECT_TO_POPULATE => $product,
        'groups' => ['product:write'],
    ],
);

$this->em->flush();
```

Seuls les champs présents dans le payload sont mis à jour. **Limite** : les collections sont **remplacées**, pas mergées — pour un merge partiel d'une collection, il faut un custom denormalizer ou un DTO dédié.

## Debug

```bash
# Liste tous les normalizers/encoders chargés, dans l'ordre de priorité
symfony console debug:container --tag=serializer.normalizer
symfony console debug:container --tag=serializer.encoder

# Affiche le metadata chargé pour une classe (groups, attributs)
symfony console debug:serializer 'App\Entity\Product'
```

`debug:serializer` (Symfony 6.3+) est le premier réflexe quand `#[Groups]` « ne marche pas » — le plus souvent, l'attribut est sur le mauvais namespace (`Symfony\Component\Serializer\Annotation\Groups` vs `Attribute\Groups`) ou l'attribut PHP n'est pas activé (projets anciens config YAML custom).

## Tests

### Test unitaire d'un normalizer custom

```php
public function test_it_normalizes_money(): void
{
    $normalizer = new MoneyNormalizer();
    $result = $normalizer->normalize(new Money(1000, 'EUR'));

    self::assertSame(['amount' => 1000, 'currency' => 'EUR'], $result);
}
```

### Test avec le Serializer réel

```php
public function test_product_serializes_with_read_group(): void
{
    $serializer = self::getContainer()->get('serializer');
    $json = $serializer->serialize(
        new Product(id: 1, name: 'Foo', priceCents: 1000),
        'json',
        ['groups' => ['product:read']],
    );

    self::assertJsonStringEqualsJsonString(
        '{"id":1,"name":"Foo","priceCents":1000}',
        $json,
    );
}
```

Préférer `assertJsonStringEqualsJsonString` à `assertSame` — l'ordre des clés JSON n'est pas garanti.

## Pièges fréquents

- **Entité Doctrine sérialisée directement** : lazy proxies rompus, relations bidirectionnelles qui bouclent, champs sensibles fuitent. Toujours passer par un DTO ou `#[Groups]` stricts.
- **`@Groups` (annotation Doctrine) au lieu de `#[Groups]` (attribut PHP)** : silencieusement ignoré si `enable_annotations` n'est plus activé. Vérifier le `use Symfony\Component\Serializer\Attribute\Groups;` — pas `Annotation\Groups`.
- **Collection vide sérialisée `{}` au lieu de `[]`** : le Serializer voit un `ArrayCollection` vide comme un objet → `{}`. Forcer `->toArray()` côté DTO ou `PRESERVE_EMPTY_OBJECTS` (à l'envers selon le sens voulu). Casse la doc OpenAPI si pas vu.
- **`DateTime` muté par erreur** : dénormaliser vers `DateTime` (pas immutable), un setter le modifie après coup côté consommateur. Toujours typer les champs en `DateTimeImmutable`.
- **`NotNormalizableValueException` en 500** : sans catch explicite, un payload JSON mal typé produit une 500. Toujours try/catch autour de `deserialize()` dans un contrôleur exposé et renvoyer 400.
- **Normalizer custom non prioritaire** : si `getSupportedTypes()` manque ou si la priorité n'est pas haute, l'`ObjectNormalizer` prend le pas et sérialise le type « à sa manière ». Vérifier avec `debug:container --tag=serializer.normalizer`.
- **Circular reference handler qui retourne l'objet entier** : un handler mal écrit re-déclenche la boucle. Retourner **toujours** un scalaire (id, iri, slug).
- **Groupes qui ne s'appliquent pas** : oubli du `'groups' => [...]` dans le contexte côté appel. Sans, tous les champs passent. Symfony ne warne pas.
- **`OBJECT_TO_POPULATE` qui remplace une collection au lieu de la merger** : comportement attendu mais contre-intuitif. Faire un normalizer custom ou utiliser un DTO intermédiaire.
- **`MAX_DEPTH_HANDLER` oublié** : avec `#[MaxDepth(2)]` mais sans handler ni `ENABLE_MAX_DEPTH => true`, le max depth est ignoré silencieusement — le graphe est sérialisé en entier.
- **Cache metadata pas chauffé en prod** : premier appel lent (reflection + parsing des attributs). `cache:warmup` à froid en déploiement — le Serializer participe au warmup automatiquement.
- **`enable_type_enforcement` mal compris** : par défaut, un payload `{"age": "42"}` dénormalisé sur un `int $age` **échoue** (type enforcement actif). Les APIs legacy qui envoient des strings partout doivent accepter ça explicitement via DTO intermédiaire, pas en désactivant la vérif.

## Déroulement

### 1 — Cadrer

- Direction : **sérialisation** (objet → format) ou **dénormalisation** (format → objet), ou les deux ?
- Format(s) cible : JSON seul, ou aussi XML / CSV / YAML ?
- Entité vs DTO : exposer directement l'entité (rarement) ou passer par un DTO (préféré en API publique) ?
- Groupes : stratégie `entité:read` / `entité:write` / scopes admin ?
- Convention de nommage externe : `camelCase`, `snake_case`, autre ? (NameConverter global ou `#[SerializedName]` par champ)
- Cas particuliers : polymorphisme (`#[DiscriminatorMap]`), édition partielle (`OBJECT_TO_POPULATE`), collections typées (PHPDoc `@var Item[]`).

### 2 — Implémenter

- DTO ou entité avec attributs `#[Groups]`, `#[SerializedName]`, `#[Context]` selon besoins.
- Contrôleur/service qui appelle `SerializerInterface::serialize()` ou `deserialize()` avec le contexte adapté.
- Try/catch `PartialDenormalizationException` + `NotNormalizableValueException` côté entrant, 400 avec détails de champs.
- Si type non-standard : normalizer custom avec `getSupportedTypes()` et tag à priorité explicite.
- `config/packages/serializer.yaml` : `name_converter` global si applicable.

### 3 — Vérifier

```bash
symfony console debug:serializer 'App\Entity\Product'           # metadata attendu ?
symfony console debug:container --tag=serializer.normalizer      # custom normalizer présent ?
vendor/bin/phpstan analyse src
```

Test : unitaire du normalizer custom, fonctionnel du contrôleur (assertJsonStringEqualsJsonString sur la réponse).

### 4 — Clôture

Afficher :

- DTO / entité + groupes déclarés.
- Format(s) supporté(s), contexte appelé (groups, options spéciales).
- Normalizer custom s'il y a lieu (classe + types gérés + priorité).
- Stratégie d'erreur côté entrant (code HTTP, payload renvoyé).
- Ce qui reste : documentation du schéma (OpenAPI manuel ou via Platform), alignement avec la validation, versionning de l'API si breaking.

## Delta Sylius

- Sylius expose ses resources via **SyliusResourceBundle** qui embarque sa propre couche de (de)sérialisation — souvent via des `Factory` / `ViewFactory` plutôt que le Serializer direct. Pour un endpoint custom au-dessus d'une resource Sylius, préférer une **view DTO** dédiée plutôt qu'un groupe `#[Groups]` posé sur la resource vendor (évite le couplage avec les mises à jour Sylius).
- Si le projet utilise **Sylius + API Platform** (l'API admin v2), aligner les groupes sur la convention Platform (`sylius:product:read`) et ne pas créer de groupes parallèles — la maintenance explose.
- Les **associations channel-scoped** (Product ↔ Channel) ne filtrent **pas** lors de la sérialisation — le Serializer dumpera tous les channels rattachés. Contourner via un DTO qui n'expose que le channel courant (injecter `ChannelContextInterface` dans un normalizer custom).
- Les **Translations Sylius** (`ProductTranslation`) sont accédées via `getName()` résolu par locale ; le Serializer les expose sous forme de tableau `{"en_US": {...}, "fr_FR": {...}}` par défaut, rarement ce qu'on veut — passer par un normalizer qui résout la locale courante.

## Argument optionnel

`/symfony:serializer-use ProductDto` — scaffolde un DTO avec groupes `product:read` / `product:write`, constructeur `readonly`, `#[Context]` sur les DateTimeImmutable. N'invente pas les champs : lit l'entité `Product` existante si elle existe.

`/symfony:serializer-use MoneyNormalizer` — scaffolde un normalizer/denormalizer custom avec `getSupportedTypes()` et tag à priorité explicite.

`/symfony:serializer-use "name_converter snake_case"` — patche `serializer.yaml` pour activer la conversion globale camelCase → snake_case, laisse `MetadataAwareNameConverter` en place pour les `#[SerializedName]`.

`/symfony:serializer-use src/Controller/ProductController.php` — audit d'un contrôleur : try/catch `PartialDenormalizationException` présent, `groups` explicites dans le contexte, pas d'entité Doctrine exposée sans filtrage, `ALLOW_EXTRA_ATTRIBUTES => false` sur les endpoints d'écriture.

`/symfony:serializer-use src/Entity/Order.php` — audit d'une entité : groupes posés, `#[Ignore]` sur les champs sensibles, `#[MaxDepth]` sur les relations bidirectionnelles, pas de fuite de champs internes.

`/symfony:serializer-use` sans argument — demande le cas (DTO entrant/sortant, normalizer custom, name converter, circular reference, audit).

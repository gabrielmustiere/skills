---
name: validation-use
description: Utilise `ValidatorInterface` Symfony hors form — DTO, payload JSON, `ConstraintViolationList`, 422 Problem Details. Déclenche sur "ValidatorInterface", "valider un DTO", "valider sans form". Impose service injecté.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /validation-use — Utiliser le Validator Symfony hors forms

> **Utilise quand** tu appelles `ValidatorInterface` hors d'un Form (DTO API, payload JSON, CLI).
> **Pas quand** tu ajoutes des contraintes sur une classe → `/symfony:validation-constraints`.
> **Pas quand** tu organises des groupes ou validations conditionnelles → `/symfony:validation-groups`.

Tu appelles directement `ValidatorInterface` depuis un contrôleur API, un service ou une commande. Tu renvoies une réponse HTTP bien formée (422 Problem Details côté API) et tu ne laisses jamais une `ConstraintViolationList` fuiter telle quelle.

## Détection préalable (obligatoire)

1. Lire `composer.json`.
2. Vérifier `symfony/validator`.
3. Si `api-platform/core` est présent → API Platform appelle le validator **automatiquement** lors du POST/PUT/PATCH sur une ressource. Pas d'appel manuel, on configure les groupes (`validationContext: ['groups' => [...]]`) dans les `#[ApiResource]` / `#[Put]`. Mentionner à l'utilisateur.
4. Si `symfony/form` est utilisé pour le flux → la validation est automatique après `handleRequest`. Ne **pas** réappeler `$validator->validate($entity)` (cf. `/symfony:form-handle`).
5. Si `symfony/serializer` présent et qu'on désérialise du JSON → combo serializer + validator est le pattern API sans form (détaillé plus bas).

## Règles fondamentales

- **Service injecté via le constructeur**, pas `$this->get('validator')`. Typehint `ValidatorInterface`.
- **`count($violations) > 0`** pour tester la présence d'erreurs, pas `null` ni cast sur `bool`. `ConstraintViolationList` est `Countable` et `IteratorAggregate`.
- **Pas de `(string) $violations` en prod** : c'est du debug, illisible côté client. Construire une réponse structurée (Problem Details, GraphQL errors, etc.).
- **Choisir les groupes** au moment de l'appel : `$validator->validate($obj, null, ['Default', 'creation'])`. Si on oublie, seul `Default` est validé (cf. `/symfony:validation-groups`).
- **Ne pas capturer `ValidationFailedException`** comme exception générique : la laisser remonter vers un listener dédié qui la convertit en réponse 422.
- **Pas de mutation après validation** : si le validator accepte l'objet, ne pas muter les propriétés en aval sans revalider — on perd la garantie.

## Pattern canonique : validation d'un DTO API

### 1 — DTO avec contraintes

```php
// src/Dto/CreateProductRequest.php
namespace App\Dto;

use Symfony\Component\Validator\Constraints as Assert;

final class CreateProductRequest
{
    public function __construct(
        #[Assert\NotBlank]
        #[Assert\Length(min: 2, max: 120)]
        public readonly string $name,

        #[Assert\PositiveOrZero]
        public readonly int $priceCents,

        #[Assert\Choice(choices: ['draft', 'published'])]
        public readonly string $status = 'draft',
    ) {}
}
```

### 2 — Désérialisation + validation dans le contrôleur

```php
use App\Dto\CreateProductRequest;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Attribute\MapRequestPayload;
use Symfony\Component\Routing\Attribute\Route;

final class ProductController extends AbstractController
{
    #[Route('/api/products', methods: ['POST'])]
    public function create(
        #[MapRequestPayload] CreateProductRequest $payload,
        ProductCreator $creator,
    ): JsonResponse {
        $product = $creator->create($payload);

        return $this->json(['id' => $product->getId()], Response::HTTP_CREATED);
    }
}
```

`#[MapRequestPayload]` (Symfony 6.3+) **désérialise et valide** automatiquement. En cas de `ValidationFailedException`, Symfony renvoie 422. Pas besoin de toucher au validator à la main dans ce cas.

### 3 — Appel manuel (sans `#[MapRequestPayload]`)

Quand `#[MapRequestPayload]` ne suffit pas (résolution de dépendances dans le DTO, contexte custom) :

```php
use Symfony\Component\Serializer\SerializerInterface;
use Symfony\Component\Validator\Validator\ValidatorInterface;

public function __construct(
    private readonly SerializerInterface $serializer,
    private readonly ValidatorInterface $validator,
) {}

public function create(Request $request): JsonResponse
{
    $payload = $this->serializer->deserialize(
        $request->getContent(),
        CreateProductRequest::class,
        'json',
    );

    $violations = $this->validator->validate($payload, groups: ['Default', 'creation']);

    if (count($violations) > 0) {
        return $this->violationsResponse($violations);
    }

    // ... logique métier
}
```

## Traiter une `ConstraintViolationList`

### Format de réponse Problem Details (RFC 7807)

```php
use Symfony\Component\Validator\ConstraintViolationListInterface;

private function violationsResponse(ConstraintViolationListInterface $violations): JsonResponse
{
    $errors = [];
    foreach ($violations as $v) {
        $errors[] = [
            'property' => $v->getPropertyPath(),
            'message'  => $v->getMessage(),
            'code'     => $v->getCode(),
        ];
    }

    return new JsonResponse([
        'type'   => 'https://example.com/errors/validation',
        'title'  => 'Validation failed',
        'status' => 422,
        'errors' => $errors,
    ], 422, ['Content-Type' => 'application/problem+json']);
}
```

Extraire ce helper dans un `ValidationErrorRenderer` partagé dès qu'on a 2+ endpoints. Ne pas le dupliquer.

### Exception Listener (recommandé)

Centraliser le rendu dans un `EventSubscriber` sur `KernelEvents::EXCEPTION` qui attrape `ValidationFailedException` → tous les endpoints renvoient le même format.

```php
final class ValidationExceptionSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [KernelEvents::EXCEPTION => 'onException'];
    }

    public function onException(ExceptionEvent $event): void
    {
        $e = $event->getThrowable();
        if (!$e instanceof ValidationFailedException) {
            return;
        }

        $event->setResponse($this->render($e->getViolations()));
    }
}
```

Depuis Symfony 7.1, `ValidationFailedException` implémente `HttpExceptionInterface` et produit 422 automatiquement — le subscriber sert à **formatter** le body, pas à fixer le code.

## Valider dans un service

Dans un `ProductCreator` appelé hors contexte HTTP (CLI, messenger handler) :

```php
use Symfony\Component\Validator\Exception\ValidationFailedException;
use Symfony\Component\Validator\Validator\ValidatorInterface;

final class ProductCreator
{
    public function __construct(
        private readonly ValidatorInterface $validator,
        private readonly EntityManagerInterface $em,
    ) {}

    public function create(CreateProductRequest $r): Product
    {
        $violations = $this->validator->validate($r);
        if (count($violations) > 0) {
            throw new ValidationFailedException($r, $violations);
        }

        $product = new Product($r->name, $r->priceCents, $r->status);
        $this->em->persist($product);
        $this->em->flush();

        return $product;
    }
}
```

Lever `ValidationFailedException` plutôt qu'une exception custom : les listeners HTTP et la couche messenger la reconnaissent.

## Valider une valeur brute (scalaire, tableau)

Hors classe, pour un seul champ.

### `Collection` pour un tableau structuré

```php
use Symfony\Component\Validator\Constraints as Assert;

$constraint = new Assert\Collection([
    'email' => [new Assert\NotBlank(), new Assert\Email()],
    'age'   => new Assert\Range(min: 0, max: 120),
]);

$violations = $this->validator->validate($data, $constraint);
```

Par défaut, les clés inconnues lèvent une violation. `allowExtraFields: true` et `allowMissingFields: true` assouplissent. À utiliser pour de la config importée (CSV, YAML utilisateur), pas pour remplacer un DTO.

### Validation Callables (tests, validation one-shot)

```php
use Symfony\Component\Validator\Validation;

$validate = Validation::createCallable(new Assert\Email());
$validate($request->query->get('email')); // ValidationFailedException si invalide

$isValid = Validation::createIsValidCallable(new Assert\Uuid());
if (!$isValid($id)) {
    throw $this->createNotFoundException();
}
```

Pratique dans une commande ou un middleware très léger. En flux principal, préférer un DTO.

## Commandes CLI

Pour valider un input utilisateur d'une `Command` :

```php
protected function execute(InputInterface $input, OutputInterface $output): int
{
    $dto = new ImportConfig(
        source: $input->getArgument('source'),
        batchSize: (int) $input->getOption('batch'),
    );

    $violations = $this->validator->validate($dto);
    if (count($violations) > 0) {
        foreach ($violations as $v) {
            $output->writeln(sprintf('<error>%s: %s</error>', $v->getPropertyPath(), $v->getMessage()));
        }
        return Command::INVALID;
    }

    // ...
    return Command::SUCCESS;
}
```

Renvoyer `Command::INVALID` (code 2), pas `FAILURE` (1), pour les erreurs de validation.

## Traduction des messages

- Domaine dédié : `translations/validators.<locale>.yaml`.
- Clés : préférer des identifiants type `product.name.blank`, pas la phrase française en dur. Les messages par défaut des contraintes Symfony sont déjà traduits dans le domaine `validators`.
- Paramètres : `#[Assert\Length(min: 2, message: 'product.name.too_short')]` → côté yaml `product.name.too_short: "Le nom doit faire au moins {{ limit }} caractères."`. Le validator fournit `{{ value }}`, `{{ limit }}`, etc.
- Pour tester la locale : `$validator->validate($obj)` respecte la locale courante (`LocaleSwitcher`) ; dans un contexte CLI sans locale, définir `$translator->setLocale('fr')` avant.

## Sévérité

`#[Assert\XXX(payload: ['severity' => 'warning'])]` transporte un payload arbitraire lisible via `$violation->getConstraint()->payload`. Permet d'afficher des violations « warning » non bloquantes côté UI.

```php
foreach ($violations as $v) {
    $severity = $v->getConstraint()->payload['severity'] ?? 'error';
    if ($severity === 'warning') {
        $warnings[] = $v;
    } else {
        $errors[] = $v;
    }
}
```

## Déroulement

### 1 — Cadrer

- Contexte : API JSON, form HTML (non, → `/symfony:form-handle`), service, CLI, messenger.
- Classe à valider : DTO (préféré) ou entité directe.
- Groupes (cf. `/symfony:validation-groups`).

### 2 — Implémenter

- DTO + contraintes (→ `/symfony:validation-constraints` pour les contraintes elles-mêmes).
- Injection de `ValidatorInterface` ou recours à `#[MapRequestPayload]`.
- Traitement structuré des `ConstraintViolation` (listener centralisé, pas de foreach dispersé).

### 3 — Vérifier

```bash
vendor/bin/phpstan analyse src
symfony console debug:validator 'App\Dto\CreateProductRequest'
```

Écrire un test functional qui POSTe un payload invalide et asserte :

```php
$client->request('POST', '/api/products', content: '{"name":"","priceCents":-1}');
self::assertResponseStatusCodeSame(422);
$body = json_decode($client->getResponse()->getContent(), true);
self::assertCount(2, $body['errors']);
```

### 4 — Clôture

Afficher :

- Endpoint / service / commande ciblé.
- DTO créé ou modifié.
- Mécanisme choisi (`#[MapRequestPayload]`, appel manuel, listener).
- Ce qui reste : traductions, contraintes supplémentaires (→ `/symfony:validation-constraints`), groupes (→ `/symfony:validation-groups`).

## Delta Sylius

- Les contrôleurs Sylius `ResourceController` valident automatiquement via la resource + form associé. Pour valider hors cycle resource (webhook, listener), injecter `ValidatorInterface` comme n'importe quel service.
- Les contraintes Sylius utilisent les groupes `['Default', 'sylius']` → passer ces groupes explicitement lors d'un appel manuel.

## Argument optionnel

`/symfony:validation-use POST /api/products CreateProductRequest` — câble la validation du payload sur un endpoint donné.

`/symfony:validation-use src/Service/ImportOrders.php` — audit d'un service qui valide des entrées (injection, groupes, exception levée).

`/symfony:validation-use` sans argument — demande le contexte (HTTP/CLI/service) et la classe à valider.

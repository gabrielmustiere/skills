---
name: controller-action
description: Écrit un contrôleur Symfony — AbstractController, helpers (render/json/redirectToRoute), mappage (#[MapQueryParameter], #[MapRequestPayload]). Déclenche sur "créer contrôleur", "action Symfony", "AbstractController". Impose contrôleur fin.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /controller-action — Écrire un contrôleur Symfony

> **Utilise quand** tu ajoutes ou révises une action HTTP qui lit une `Request` et renvoie une `Response` (HTML, JSON, fichier, stream, redirection).
> **Pas quand** tu veux juste poser/ajuster du routing (path, methods, requirements, `host`, `schemes`, génération d'URL) → `/symfony:routing-define` couvre tout le registre `#[Route]`.
> **Pas quand** tu fais du CRUD sur une Resource Sylius — le `ResourceController` vendor gère index/create/update/show/delete.

Tu crées ou révises un contrôleur Symfony. Un contrôleur est une fonction PHP qui lit une `Request` et renvoie une `Response` — rien d'autre. Tu fais passer tout le métier par un service et tu t'appuies sur les helpers de `AbstractController` plutôt que de réinventer l'accès à Twig, au routeur, au security, à la session.

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine du projet.
2. Vérifier `symfony/framework-bundle` dans les dépendances.
   - Présent → OK, continuer.
   - Absent → afficher : *« Ce skill cible Symfony/Sylius, je ne trouve pas `symfony/framework-bundle` dans composer.json. On continue quand même ou on change d'approche ? »* et attendre la réponse.
3. Repérer la convention de routing du projet : attributs PHP `#[Route]` (standard Symfony 6+) ou YAML/XML sous `config/routes/` (legacy, à ne pas introduire dans un projet neuf).
4. Si `sylius/sylius` est présent → signaler en une ligne que Sylius expose un `ResourceController` générique pour le CRUD des Resources ; un contrôleur custom n'est justifié que pour une action hors CRUD (dashboard, webhook, export).

## Règles fondamentales

- **Étendre `AbstractController`** : donne accès à `render()`, `json()`, `redirectToRoute()`, `createNotFoundException()`, `addFlash()`, `getUser()`, `denyAccessUnlessGranted()`, `isGranted()`, sans les injecter à la main. Pas besoin d'extender `Controller` (déprécié) ni d'implémenter quoi que ce soit.
- **Toujours renvoyer une `Response`** (ou une sous-classe : `JsonResponse`, `RedirectResponse`, `BinaryFileResponse`, `StreamedResponse`). Une méthode qui ne renvoie pas de `Response` casse le kernel, sauf à déclencher un event `kernel.view`.
- **Routing par attribut `#[Route]`** sur la méthode publique. Le préfixe commun (URL ou nom) va sur un `#[Route]` au niveau de la classe. Toujours nommer la route (`name: 'product_show'`) — les URLs se construisent par nom, pas par chaîne.
- **Méthodes HTTP explicites** : `methods: ['GET']`, `methods: ['POST']`, etc. Ne jamais laisser une action CRUD accepter `ANY`. Pour un form, `methods: ['GET', 'POST']` (affichage + soumission, cf. `/symfony:form-handle`).
- **Contrôleur fin, service pour le métier** : le contrôleur orchestre (valider les inputs → appeler un service → choisir la réponse). Aucune requête Doctrine, aucun calcul métier, aucun appel HTTP externe directement dans le contrôleur. Déléguer à un service (`/symfony:service-define`).
- **Injection par argument de méthode** pour les services spécifiques à une action ; **injection constructeur** pour les services partagés par plusieurs actions de la même classe. Ne pas injecter l'EntityManager dans un contrôleur — passer par un repository ou un service applicatif.
- **`declare(strict_types=1)`** en tête de chaque fichier, types de retour explicites (`: Response`, `: JsonResponse`).
- **Pas de `$this->container` ni de `getContainer()`** : anti-pattern. Tout ce qui serait tiré du container est déjà exposé par `AbstractController` ou injectable.
- **Actions `final`** : un contrôleur n'a pas vocation à être étendu. Marquer la classe `final`.

## Squelette canonique

```php
// src/Controller/ProductController.php
declare(strict_types=1);

namespace App\Controller;

use App\Service\ProductFinder;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

#[Route('/product', name: 'product_')]
final class ProductController extends AbstractController
{
    public function __construct(
        private readonly ProductFinder $finder,
    ) {}

    #[Route('/{id}', name: 'show', methods: ['GET'], requirements: ['id' => '\d+'])]
    public function show(int $id): Response
    {
        $product = $this->finder->find($id) ?? throw $this->createNotFoundException();

        return $this->render('product/show.html.twig', [
            'product' => $product,
        ]);
    }
}
```

Le contrôleur est enregistré automatiquement comme service (resource `App\:` de `services.yaml`, cf. `/symfony:service-define`). L'autowiring résout `ProductFinder` via le constructeur.

## Paramètres d'action

### Route placeholders

```php
#[Route('/article/{slug}', name: 'article_show', methods: ['GET'])]
public function show(string $slug): Response { /* ... */ }
```

Le type PHP convertit automatiquement (`int`, `string`, `bool`, `\DateTimeImmutable` via value resolver).

### `#[MapEntity]` (implicite via type-hint)

```php
#[Route('/product/{id}', name: 'product_show', methods: ['GET'])]
public function show(Product $product): Response
{
    return $this->render('product/show.html.twig', ['product' => $product]);
}
```

`doctrine/doctrine-bundle` requis. Par défaut, matche sur `{id}`. Pour un slug ou un critère custom → `#[MapEntity(mapping: ['slug' => 'slug'])]`, détails dans `/symfony:doctrine-query`.

### `#[MapQueryParameter]` (Symfony 6.3+)

```php
use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;

#[Route('/search', name: 'search', methods: ['GET'])]
public function search(
    #[MapQueryParameter] string $q = '',
    #[MapQueryParameter] int $page = 1,
): Response {
    // $q et $page viennent de la query string, typés, avec défauts.
}
```

Remplace `$request->query->get('q')` + cast manuel. Les contraintes de validation (`#[Assert\Length]`, `#[Assert\Positive]`) s'appliquent sur le paramètre.

### `#[MapQueryString]` — bind la query string sur un DTO

```php
final class SearchCriteria
{
    public function __construct(
        public string $q = '',
        #[Assert\Positive] public int $page = 1,
    ) {}
}

#[Route('/search', name: 'search', methods: ['GET'])]
public function search(#[MapQueryString] SearchCriteria $criteria): Response { /* ... */ }
```

### `#[MapRequestPayload]` — bind le body JSON/form sur un DTO

```php
final class CreateProductRequest
{
    public function __construct(
        #[Assert\NotBlank] public string $name,
        #[Assert\PositiveOrZero] public int $priceCents,
    ) {}
}

#[Route('/api/products', name: 'api_product_create', methods: ['POST'])]
public function create(
    #[MapRequestPayload] CreateProductRequest $payload,
    ProductCreator $creator,
): JsonResponse {
    $product = $creator->create($payload);

    return $this->json($product, Response::HTTP_CREATED);
}
```

Valide automatiquement (400 si contraintes KO), dispensant d'un FormType pour les API JSON.

### `Request` brut

À utiliser quand l'action lit plusieurs champs hétérogènes ou les headers :

```php
public function action(Request $request): Response
{
    $ua   = $request->headers->get('User-Agent');
    $body = $request->getContent();
    $ip   = $request->getClientIp();
    // ...
}
```

Préférer `#[MapQueryParameter]` / `#[MapRequestPayload]` quand le format est connu.

## Helpers d'`AbstractController`

| Besoin                          | Helper                                             |
|---------------------------------|----------------------------------------------------|
| Rendre un template Twig         | `$this->render('path.html.twig', [...])`           |
| Réponse JSON                    | `$this->json($data, $status)`                      |
| Redirection vers une route      | `$this->redirectToRoute('name', [...], $status)`   |
| Redirection vers une URL brute  | `$this->redirect($url, $status)`                   |
| 404                             | `throw $this->createNotFoundException('msg?')`     |
| 403                             | `throw $this->createAccessDeniedException('msg?')` |
| Check droit                     | `$this->denyAccessUnlessGranted('ROLE', $subject)` |
| Utilisateur connecté            | `$this->getUser()`                                 |
| Message flash                   | `$this->addFlash('success', 'key')`                |
| Fichier téléchargé              | `$this->file($path, $name)`                        |
| Stream                          | `new StreamedResponse(callable)`                   |
| Forward (interne)               | `$this->forward('OtherController::action', [...])` |

## Cas particuliers

### API JSON

```php
#[Route('/api/products/{id}', name: 'api_product_show', methods: ['GET'])]
public function show(int $id, ProductFinder $finder): JsonResponse
{
    $product = $finder->find($id) ?? throw $this->createNotFoundException();

    return $this->json($product, context: ['groups' => ['product:read']]);
}
```

Les `groups` s'appuient sur le Serializer (cf. `/symfony:serializer-use`). Pour les gros volumes, préférer API Platform.

### Téléchargement fichier

```php
use Symfony\Component\HttpFoundation\ResponseHeaderBag;

#[Route('/invoice/{id}/pdf', name: 'invoice_pdf', methods: ['GET'])]
public function pdf(Invoice $invoice, InvoiceRenderer $renderer): Response
{
    $path = $renderer->toTempPdf($invoice);

    return $this->file($path, "facture-{$invoice->getNumber()}.pdf", ResponseHeaderBag::DISPOSITION_ATTACHMENT);
}
```

### Stream (CSV, SSE, long download)

```php
use Symfony\Component\HttpFoundation\StreamedResponse;

#[Route('/export/orders.csv', name: 'orders_export', methods: ['GET'])]
public function export(OrderExporter $exporter): StreamedResponse
{
    $response = new StreamedResponse(function () use ($exporter): void {
        $exporter->writeCsv(fopen('php://output', 'wb'));
    });

    $response->headers->set('Content-Type', 'text/csv; charset=UTF-8');
    $response->headers->set('Content-Disposition', 'attachment; filename="orders.csv"');

    return $response;
}
```

### Redirection externe

```php
return $this->redirect('https://example.com', Response::HTTP_MOVED_PERMANENTLY);
```

N'utiliser `redirect()` que pour les URLs externes. Pour une route interne : `redirectToRoute()` (refactor-safe).

### Contrôleur invocable (une seule action)

```php
#[Route('/health', name: 'health', methods: ['GET'])]
final class HealthController extends AbstractController
{
    public function __invoke(): JsonResponse
    {
        return $this->json(['status' => 'ok']);
    }
}
```

Pratique pour les actions uniques (health-check, webhook). Le nom de classe suffit en routing.

### Sécurité d'une action

```php
use Symfony\Component\Security\Http\Attribute\IsGranted;

#[Route('/admin/products', name: 'admin_product_index', methods: ['GET'])]
#[IsGranted('ROLE_ADMIN')]
public function index(): Response { /* ... */ }
```

Préférer l'attribut `#[IsGranted]` à `denyAccessUnlessGranted()` quand la règle est statique. Pour une règle liée à l'objet (`#[IsGranted('edit', subject: 'product')]`), passer par un Voter.

### Session / CSRF

```php
$session = $request->getSession();             // depuis la Request
$this->isCsrfTokenValid('delete'.$id, $token); // helper AbstractController
```

Ne pas injecter `SessionInterface` directement — l'obtenir depuis la `Request`.

## Erreurs et codes HTTP

- 404 ressource absente → `throw $this->createNotFoundException()`.
- 403 droit manquant → `throw $this->createAccessDeniedException()` (ou `#[IsGranted]` en amont).
- 400 payload invalide → `#[MapRequestPayload]` lève automatiquement une `HttpException 422` si validation KO, ou `400` si JSON malformé. Pour custom : `throw new BadRequestHttpException('...')`.
- 410 ressource supprimée → `throw new HttpException(Response::HTTP_GONE)`.
- 500 → laisser fuiter l'exception métier, Symfony log et rend la page d'erreur. Ne pas try/catch pour renvoyer un 500 "manuel".

## Tests

Un test fonctionnel par action publique :

```php
public function testShowReturnsProductPage(): void
{
    $client = static::createClient();
    $client->request('GET', '/product/42');

    self::assertResponseIsSuccessful();
    self::assertSelectorTextContains('h1', 'Widget');
}

public function testShowReturns404WhenMissing(): void
{
    $client = static::createClient();
    $client->request('GET', '/product/99999');

    self::assertResponseStatusCodeSame(404);
}
```

Les tests de form → `/symfony:form-handle`. Les tests d'API JSON → assertions sur `$client->getResponse()->getContent()` + `json_decode`.

## Pièges fréquents

- **Oubli du `return`** : une action qui ne retourne rien renvoie `null` → `LogicException: The controller must return a "Response" object`.
- **`redirectToRoute` avec une route inexistante** : échec silencieux au build d'URL → `RouteNotFoundException`. Toujours utiliser le nom déclaré dans `#[Route(name: ...)]`.
- **`methods:` oublié** : l'action accepte GET/POST/PUT/DELETE/… par défaut, ouvrant des surfaces non voulues (CSRF GET, mutations idempotentes). Toujours borner.
- **Injection d'`EntityManagerInterface`** dans le contrôleur : symptôme d'une logique métier qui devrait être dans un service. Déplacer.
- **`render()` avec un FormType** : passer `$form` (et non `$form->createView()` depuis Symfony 6.2). Voir `/symfony:form-handle`.
- **Slash final dans `#[Route('/product/')]`** : Symfony traite `/product` et `/product/` comme différents. Choisir une convention (sans slash final) et la maintenir.
- **`@Route` (annotations)** dans un projet Symfony 6+ : `doctrine/annotations` a été retiré des dépendances Flex. Utiliser les attributs PHP natifs `#[Route]`.
- **Contrôleur enregistré manuellement dans `services.yaml`** : inutile, le resource `App\:` couvre `src/Controller/`. Un contrôleur n'a pas besoin de `public: true` (le `ControllerResolver` le détecte via le tag `controller.service_arguments` posé par `autoconfigure`).

## Vérifications

```bash
symfony console debug:router                              # toutes les routes
symfony console debug:router product_show                 # détail d'une route
symfony console router:match /product/42                  # quelle route pour cette URL
symfony console debug:container App\\Controller\\ProductController
vendor/bin/phpstan analyse src/Controller
```

## Delta Sylius

- Pour les Resources Sylius (`Product`, `Order`, `Customer`…), **ne pas réécrire** un contrôleur CRUD : le `ResourceController` de `sylius/resource-bundle` gère `index`/`create`/`update`/`show`/`delete` avec templates, repository, factory configurables dans `_sylius.yaml`.
- Un contrôleur custom est légitime pour : dashboard admin, export, webhook, intégration externe, endpoint API non resource.
- Étendre `Sylius\Bundle\ResourceBundle\Controller\ResourceController` uniquement pour surcharger finement une action d'une resource ; sinon la configuration suffit.
- Pour une action publique d'une Resource Sylius, le routing est déjà posé par le bundle — vérifier `symfony console debug:router | grep sylius_`.

## Déroulement

### 1 — Cadrer

- URL + méthode HTTP ciblée, nom de route.
- Inputs : placeholders, query, body JSON, form, fichier ? Mapping avec `#[MapEntity]` / `#[MapQueryParameter]` / `#[MapRequestPayload]` ou `Request` brut ?
- Output : HTML (Twig), JSON, fichier, stream, redirection ?
- Sécurité : anonyme, authentifié, rôle, voter ?

### 2 — Implémenter

- Classe `final` sous `src/Controller/`, extends `AbstractController`, `declare(strict_types=1)`.
- `#[Route]` au niveau classe pour préfixe + nom, `#[Route]` sur chaque méthode.
- Corps de l'action : ≤ 10 lignes en général. Si plus → le métier est à extraire.
- Dépendances par constructeur (partagées) ou argument de méthode (spécifiques).

### 3 — Vérifier

- `symfony console debug:router` la route apparaît.
- Appel manuel (`curl`, navigateur, `symfony console router:match`).
- Test functional a minima (200/302 selon le cas + 404 si applicable).

### 4 — Clôture

Afficher :

- Fichier créé/modifié, route(s) déclarée(s), méthode HTTP.
- Helpers/mappers utilisés (`#[MapRequestPayload]`, `#[IsGranted]`, …).
- Ce qui reste : template Twig (`/symfony:form-render`), service applicatif (`/symfony:service-define`), sérialisation API (`/symfony:serializer-use`), traitement de form (`/symfony:form-handle`), tests fonctionnels.

## Argument optionnel

`/symfony:controller-action ProductController show` — scaffolde l'action `show` dans `ProductController` (classe créée si absente).

`/symfony:controller-action src/Controller/OrderController.php` — audit du fichier (routes, méthodes HTTP bornées, logique métier déléguée, helpers utilisés, sécurité).

`/symfony:controller-action` sans argument — demande l'URL, la méthode HTTP et le comportement attendu.

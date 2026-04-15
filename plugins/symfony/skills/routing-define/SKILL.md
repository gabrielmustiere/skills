---
name: routing-define
description: Définit le routing Symfony — #[Route] (path, methods, requirements, schemes, stateless, host, locale), génération d'URL (UrlGenerator, path/url Twig), URIs signées. Déclenche sur "créer route", "#[Route]", "générer URL". Impose routes nommées.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /routing-define — Définir le routing Symfony

> **Utilise quand** tu poses, renommes, contraintes ou localises une route, tu configures `default_uri` pour la génération hors requête, ou tu signes une URL à durée limitée.
> **Pas quand** tu écris le corps d'une action (lecture de `Request`, `#[MapRequestPayload]`, helpers d'`AbstractController`) → `/symfony:controller-action` couvre ce registre.
> **Pas quand** tu ajoutes juste un `#[Route]` trivial sur une nouvelle action — `/symfony:controller-action` inclut le minimum syndical.

Tu déclares une ou plusieurs routes Symfony et tu t'assures qu'elles sont atteignables, typées, nommées, et référencées par nom partout où une URL est construite. Le routing Symfony ne se limite pas à un path — il porte aussi les méthodes HTTP acceptées, les contraintes sur les placeholders, l'environnement, le schéma, l'hôte, la condition d'activation et la génération d'URL à l'envers.

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine du projet.
2. Vérifier `symfony/framework-bundle` dans les dépendances.
   - Présent → OK, continuer.
   - Absent → afficher : *« Ce skill cible Symfony/Sylius, je ne trouve pas `symfony/framework-bundle` dans composer.json. On continue quand même ou on change d'approche ? »* et attendre la réponse.
3. Repérer la convention de routing du projet :
   - `config/routes/attributes.yaml` (ou équivalent) + `#[Route]` sur les contrôleurs → standard Symfony 6+.
   - `config/routes.yaml` avec des entrées YAML → legacy mais encore supporté ; ne pas mélanger les deux styles sans raison.
   - Annotations `@Route` (Doctrine) → supprimées en Symfony 6.0, migrer vers les attributs PHP.
4. Lire `config/packages/routing.yaml` pour repérer `default_uri` (nécessaire pour générer des URLs depuis un worker, une commande, un handler Messenger).
5. Si `sylius/sylius` est présent → les routes des Resources sont déjà posées par `sylius/resource-bundle`. Vérifier `bin/console debug:router | grep sylius_` avant de dupliquer une route CRUD.

## Règles fondamentales

- **Attributs PHP `#[Route]`** sur les contrôleurs : co-localisation route + action, analyse statique, aucune duplication YAML. Réserver YAML aux routes "configuration" (template direct, redirect controller, routes d'un bundle tiers montées à plusieurs endroits).
- **Toujours nommer la route** (`name: 'product_show'`). La génération d'URL passe par le nom, jamais par une chaîne littérale. Une URL en dur dans du code = dette.
- **Méthodes HTTP explicites** (`methods: ['GET']`, `['POST']`, `['GET', 'POST']`). Une route sans `methods` accepte tout — surface d'attaque et faux positifs sur les mutations idempotentes.
- **Placeholders typés + `requirements`** : `{id}` seul matche n'importe quoi. Ajouter `requirements: ['id' => '\d+']` (ou l'enum `Requirement::DIGITS`) pour borner. Pour une UUID : `Requirement::UUID`. Préférer les constantes de `Symfony\Component\Routing\Requirement\Requirement` aux regex magiques.
- **Préfixes au niveau classe** : `#[Route('/product', name: 'product_')]` sur la classe, puis `#[Route('/{id}', name: 'show', methods: ['GET'])]` sur les méthodes. Les noms se concatènent → `product_show`. Évite la duplication du préfixe URL et du préfixe de nom.
- **Pas de slash final** dans les paths (`/product`, pas `/product/`). Choisir la convention sans slash final et s'y tenir — Symfony traite `/product` et `/product/` comme deux routes différentes.
- **Stateless par défaut pour les API** : `stateless: true` déclenche une exception si le handler touche à la session, ce qui protège le caching HTTP en amont.
- **Schéma explicite pour l'auth** : `schemes: ['https']` sur les routes de login, paiement, callbacks OAuth. Couplé à `trusted_proxies`, garantit qu'une redirection HTTP → HTTPS est déclenchée côté framework.
- **Priority seulement en dernier recours** : si deux routes se recouvrent (`/blog/{slug}` vs `/blog/list`), la bonne réponse est de déclarer la plus spécifique d'abord ou d'ajouter un `requirements` qui les désambiguïse. `priority: 2` marche mais masque le chevauchement.
- **`#[Route]` sur une classe `final`** : un contrôleur n'a pas à être étendu (cf. `/symfony:controller-action`).
- **Un path, une responsabilité** : une action par path + méthode. Pas de `switch ($request->getMethod())` dans un handler — séparer en deux actions avec `methods: ['GET']` et `methods: ['POST']`.

## Squelette canonique

```php
// src/Controller/ProductController.php
declare(strict_types=1);

namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;
use Symfony\Component\Routing\Requirement\Requirement;

#[Route('/product', name: 'product_')]
final class ProductController extends AbstractController
{
    #[Route('', name: 'index', methods: ['GET'])]
    public function index(): Response { /* ... */ }

    #[Route('/{id}', name: 'show', methods: ['GET'], requirements: ['id' => Requirement::DIGITS])]
    public function show(int $id): Response { /* ... */ }

    #[Route('/{slug}', name: 'show_by_slug', methods: ['GET'], requirements: ['slug' => '[a-z0-9-]+'])]
    public function showBySlug(string $slug): Response { /* ... */ }
}
```

Routes résultantes : `product_index` (`GET /product`), `product_show` (`GET /product/42`), `product_show_by_slug` (`GET /product/widget-xl`). L'ordre de déclaration fixe la priorité — `show` (digits) passe avant `show_by_slug` (slug alphanum).

## Placeholders et contraintes

### Requirements via constantes

```php
use Symfony\Component\Routing\Requirement\Requirement;

#[Route('/order/{uuid}', name: 'order_show', requirements: ['uuid' => Requirement::UUID])]
#[Route('/asset/{slug}', requirements: ['slug' => Requirement::ASCII_SLUG])]
#[Route('/page/{page}', requirements: ['page' => Requirement::POSITIVE_INT])]
#[Route('/file/{path}', requirements: ['path' => Requirement::CATCH_ALL])] // .+  (autorise les /)
```

`Requirement::*` est plus lisible qu'une regex et évite les erreurs classiques (`\d+` refuse `0`, pas `[1-9]\d*`).

### Syntaxe inline

```php
#[Route('/blog/{page<\d+>?1}', name: 'blog_list')]
public function list(int $page): Response { /* $page défaut 1 */ }
```

Format `{nom<regex>?défaut}`. Concis pour les cas simples, mais illisible au-delà. Préférer le tableau `requirements` + valeur par défaut dans l'argument dès qu'il y a 2 placeholders.

### Forcer un slash dans un placeholder

```php
#[Route('/share/{token}', requirements: ['token' => '.+'])]
public function share(string $token): Response { /* token peut contenir des / */ }
```

Attention : un placeholder permissif en fin de path peut ensuite casser la reconnaissance d'un `{_format}`. Préférer `requirements: ['token' => '[^/]+']` sauf nécessité.

### Backed enum en placeholder

```php
use App\Enum\OrderStatus;

#[Route('/orders/{status}', name: 'orders_by_status', methods: ['GET'])]
public function byStatus(OrderStatus $status): Response { /* ... */ }
```

Symfony résout automatiquement la valeur de l'URL en instance d'enum (depuis 6.4). URL invalide → 404 sans code custom.

## Méthodes HTTP, schéma, hôte

### Méthodes multiples

```php
#[Route('/profile/edit', name: 'profile_edit', methods: ['GET', 'POST'])]
public function edit(Request $request): Response { /* form handle, cf. /symfony:form-handle */ }
```

### HTTPS obligatoire

```php
#[Route('/login', name: 'login', methods: ['GET', 'POST'], schemes: ['https'])]
public function login(): Response { /* ... */ }
```

En dev HTTP pur, Symfony refusera de générer l'URL (ou la matchera en `redirect` vers HTTPS). Configurer `trusted_proxies` / `trusted_headers` pour que le reverse proxy serve correctement le schéma.

### Sous-domaine

```php
#[Route('/', name: 'mobile_home', host: 'm.{domain}', defaults: ['domain' => 'example.com'], requirements: ['domain' => 'example\.com|example\.fr'])]
public function mobileHome(): Response { /* ... */ }
```

`host` peut contenir des placeholders. Utile pour multi-tenant par sous-domaine — mais attention, le placeholder `{domain}` rentre dans la génération d'URL.

### Condition (Expression Language)

```php
#[Route(
    '/api/beta/search',
    name: 'api_beta_search',
    condition: "context.getMethod() in ['GET', 'HEAD'] and request.headers.get('X-Beta') == 'true'"
)]
public function search(): Response { /* ... */ }
```

Dernier recours pour du matching complexe (feature flag, user-agent, header custom). Coûte une évaluation à chaque requête — préférer un Voter ou un middleware (`kernel.controller` listener) quand c'est du contrôle d'accès.

## Environnement, alias, stateless

### Route dev-only

```php
#[Route('/_debug/dump-config', name: '_debug_dump_config', env: 'dev')]
public function dumpConfig(): Response { /* ... */ }
```

Invisible en prod — pas de risque d'oubli de firewall. Préférer au classique `if ('dev' !== $this->kernel->getEnvironment())`.

### Alias (migration de nom)

```php
#[Route('/product/{id}', name: 'product_details', alias: ['product_show'])]
public function show(int $id): Response { /* ... */ }
```

Permet de renommer une route sans casser les appels existants à `redirectToRoute('product_show')` / `path('product_show')`. Pour pousser à la migration :

```php
use Symfony\Component\Routing\Attribute\DeprecatedAlias;

#[Route('/product/{id}', name: 'product_details', alias: [new DeprecatedAlias('product_show', 'app/core', '2.1')])]
```

Symfony émet une deprecation à chaque utilisation du vieux nom.

### Stateless

```php
#[Route('/api/ping', name: 'api_ping', methods: ['GET'], stateless: true)]
public function ping(): JsonResponse { return $this->json(['pong' => true]); }
```

Toute lecture de session dans le handler lève `UnexpectedSessionUsageException`. À poser systématiquement sur les routes d'API publiques.

## Groupes et préfixes

### Attributs au niveau classe

```php
#[Route('/admin', name: 'admin_', requirements: ['_locale' => 'en|fr'])]
final class AdminController extends AbstractController
{
    #[Route('/dashboard', name: 'dashboard', methods: ['GET'])]
    public function dashboard(): Response { /* admin_dashboard — /admin/dashboard */ }
}
```

Les attributs `requirements`, `defaults`, `schemes`, `methods`, `host`, `condition` posés au niveau classe s'appliquent à toutes les méthodes.

### Import YAML avec préfixe

```yaml
# config/routes/admin.yaml
admin:
    resource:
        path: ../../src/Controller/Admin/
        namespace: App\Controller\Admin
    type: attribute
    prefix: /admin
    name_prefix: admin_
    schemes: [https]
    trailing_slash_on_root: false
```

À utiliser quand un même contrôleur doit être monté deux fois (ex : `/admin/product` et `/vendor/{vendorId}/product`). Sinon, préférer l'attribut au niveau classe.

## Routes localisées (i18n)

```php
#[Route(
    path: [
        'en' => '/about-us',
        'fr' => '/a-propos',
        'nl' => '/over-ons',
    ],
    name: 'about',
    methods: ['GET'],
)]
public function about(): Response { /* ... */ }
```

Trois routes générées : `about.en`, `about.fr`, `about.nl`. L'injection de `$_locale` (ou l'usage de `$request->getLocale()`) fonctionne sans placeholder explicite. Génération :

```twig
{{ path('about', {_locale: 'fr'}) }}  {# → /a-propos #}
```

Pour un préfixe de locale systématique (`/en/...`, `/fr/...`), préférer un import YAML avec `prefix: { en: '', fr: '/fr' }`.

## Génération d'URL

### Depuis un contrôleur (`AbstractController`)

```php
$this->generateUrl('product_show', ['id' => 42]);                 // /product/42
$this->generateUrl('product_show', ['id' => 42], UrlGeneratorInterface::ABSOLUTE_URL); // https://example.com/product/42
$this->redirectToRoute('product_show', ['id' => 42], 302);
```

### Depuis un service (injection)

```php
use Symfony\Component\Routing\Generator\UrlGeneratorInterface;

final class InvoiceMailer
{
    public function __construct(private readonly UrlGeneratorInterface $urls) {}

    public function send(Invoice $invoice): void
    {
        $link = $this->urls->generate(
            'invoice_show',
            ['id' => $invoice->getId()],
            UrlGeneratorInterface::ABSOLUTE_URL,
        );
        // ... envoi mail
    }
}
```

### Depuis Twig

```twig
<a href="{{ path('product_show', {id: product.id}) }}">{{ product.name }}</a>
<link rel="canonical" href="{{ url('product_show', {id: product.id}) }}">
```

`path()` = relatif, `url()` = absolu. Un paramètre qui n'existe pas dans la définition de la route est ajouté en query string.

### Depuis un worker / une commande / un handler Messenger

Aucune `Request` en contexte → `UrlGeneratorInterface::ABSOLUTE_URL` tombe sur `http://localhost` par défaut. **Configurer `default_uri`** :

```yaml
# config/packages/routing.yaml
framework:
    router:
        default_uri: 'https://example.com'
```

Sans cette config, les URLs dans les mails envoyés depuis un worker pointent vers localhost — classique régression en prod.

### Modes de génération

| Constante                                | Résultat                            |
|------------------------------------------|-------------------------------------|
| `ABSOLUTE_PATH` (défaut)                 | `/product/42`                       |
| `ABSOLUTE_URL`                           | `https://example.com/product/42`    |
| `NETWORK_PATH`                           | `//example.com/product/42`          |
| `RELATIVE_PATH`                          | `../product/42`                     |

## URIs signées

Pour un lien à usage limité (validation email, téléchargement temporaire, webhook callback) :

```php
use Symfony\Component\HttpFoundation\UriSigner;

final class ConfirmationLinkBuilder
{
    public function __construct(
        private readonly UrlGeneratorInterface $urls,
        private readonly UriSigner $signer,
    ) {}

    public function build(User $user): string
    {
        $url = $this->urls->generate('user_confirm', ['id' => $user->getId()], UrlGeneratorInterface::ABSOLUTE_URL);

        return $this->signer->sign($url, new \DateInterval('PT1H')); // valide 1h
    }
}
```

Côté contrôleur de réception :

```php
use Symfony\Component\HttpKernel\Attribute\IsSignatureValid;

#[Route('/user/{id}/confirm', name: 'user_confirm', methods: ['GET'])]
#[IsSignatureValid]
public function confirm(int $id): Response { /* ... */ }
```

L'attribut `#[IsSignatureValid]` lève 403 si la signature est invalide ou expirée. Ne pas rouler sa propre vérification `hash_equals` — le composant gère les constants-time comparisons et l'expiration.

## Debugging

```bash
symfony console debug:router                              # toutes les routes (nom, méthode, schéma, path)
symfony console debug:router product_show                 # détail d'une route (regex compilée, defaults, requirements)
symfony console debug:router --show-aliases               # inclut les alias
symfony console debug:router --show-controllers           # inclut le contrôleur cible
symfony console debug:router --method=POST                # filtre par méthode
symfony console router:match /product/42                  # quelle route matche cette URL, et pourquoi les autres ne matchent pas
symfony console router:match /product/42 --method=POST    # match avec méthode
symfony console cache:clear                               # après ajout/suppression de route
```

`router:match` est l'outil décisif quand une URL renvoie 404 inexplicable — il liste chaque route candidate et affiche la raison du rejet (path, method, host, condition).

## Pièges fréquents

- **Slash final incohérent** : `/product` vs `/product/` — 404 silencieux. Choisir sans slash final, être cohérent.
- **`methods:` oublié** : l'action accepte `GET|POST|PUT|DELETE|...` par défaut. Formulaires exposés en GET, mutations idempotentes, surface CSRF élargie.
- **URL en dur** : `return $this->redirect('/product/'.$id)` → casse dès qu'on déplace le préfixe. Toujours `redirectToRoute('product_show', ['id' => $id])`.
- **Paramètre manquant à la génération** : `MissingMandatoryParametersException` — la route a un placeholder non fourni. `symfony console debug:router <nom>` affiche les placeholders requis.
- **Ordre des routes** : `/{slug}` déclarée avant `/list` catch tout. Déclarer le spécifique avant le générique, ou ajouter un `requirements` qui exclut l'autre.
- **`host` sans `trusted_hosts`** : en prod, Symfony refuse de matcher si `framework.trusted_hosts` n'autorise pas l'hôte. Symptôme : 404 sur toutes les routes après passage en prod derrière un domaine custom.
- **Annotations `@Route`** dans un projet 6+ : `doctrine/annotations` a été retiré. Migrer vers les attributs PHP (`php bin/console make:controller` génère déjà des `#[Route]`).
- **URL absolue dans un worker sans `default_uri`** : les mails transactionnels linkent vers `http://localhost`. Configurer `framework.router.default_uri`.
- **Condition avec un service non existant** : `service('foo')` dans une `condition` crash au boot. L'expression est compilée au premier match — `lint:container` ne le détecte pas toujours.
- **Routes localisées sans `_locale` en `defaults`** au niveau groupe : le matching marche mais la génération d'URL demande `_locale` explicitement à chaque `path()`.
- **`stateless: true` + `getUser()`** : dans certains firewalls, `getUser()` hydrate la session. Vérifier le firewall stateless (`stateless: true` sur la firewall config aussi) si l'exception tombe.

## Vérifications

```bash
symfony console debug:router
symfony console router:match <url>
symfony console lint:yaml config/routes/
symfony console cache:clear
vendor/bin/phpstan analyse src/Controller
```

Pour un smoke test rapide d'un ensemble de routes :

```bash
for route in product_index product_show product_create; do
    symfony console debug:router "$route" > /dev/null 2>&1 && echo "OK $route" || echo "KO $route"
done
```

## Delta Sylius

- Les Resources Sylius (`sylius.product`, `sylius.order`, …) génèrent automatiquement les routes CRUD via `sylius/resource-bundle`. Nommage : `sylius_admin_product_index`, `sylius_shop_product_show`. **Vérifier avant d'en créer une** : `bin/console debug:router | grep sylius_<resource>`.
- Pour surcharger une route Sylius (changer le path, ajouter une contrainte), passer par `config/routes/sylius_*.yaml` ou `_sylius.yaml` côté resource config — pas par un nouveau `#[Route]` qui entrerait en conflit.
- Les routes custom (dashboard admin, webhook paiement, export) vivent sous `App\Controller\` comme dans tout projet Symfony, et suivent la convention de nommage du projet (souvent `app_admin_<action>` ou `app_shop_<action>`).
- Attention au `trailing_slash_on_root` de l'import YAML Sylius : certains templates admin génèrent `/admin/`, d'autres `/admin` — les deux doivent résoudre. Le `RedirectableUrlMatcher` de Symfony s'en occupe si la config le laisse faire.

## Déroulement

### 1 — Cadrer

- Path + méthode HTTP + nom de route.
- Placeholders : type, contrainte (`Requirement::*`), valeur par défaut éventuelle.
- Portée : env dev-only ? schema HTTPS ? stateless ? localisée ? sous-domaine ?
- Où cette route est-elle générée (Twig, service, mail, API externe) ? Si externe → `default_uri` configuré.

### 2 — Implémenter

- `#[Route]` sur le contrôleur (préfixe classe + attribut méthode) OU entrée YAML pour un cas "configuration".
- Toujours : `name`, `methods`, `requirements` sur les placeholders non triviaux.
- Si besoin de génération asynchrone (worker, commande) → vérifier `framework.router.default_uri`.

### 3 — Vérifier

- `bin/console debug:router <name>` → route présente, path + method + requirements corrects.
- `bin/console router:match <url>` → URL testée matche bien cette route.
- `bin/console cache:clear` si la route n'apparaît pas.
- `curl -I <url>` (ou navigateur) → 200/302/404 attendu.

### 4 — Clôture

Afficher :

- Fichier touché, nom(s) de route, path, méthode(s).
- Contraintes posées (`requirements`, `schemes`, `stateless`, `env`, `host`).
- Ce qui reste : action du contrôleur (`/symfony:controller-action`), form (`/symfony:form-handle`), sérialisation API (`/symfony:serializer-use`), sécurité (voter, firewall), tests fonctionnels.

## Argument optionnel

`/symfony:routing-define ProductController show /product/{id} GET` — scaffolde la route `product_show` dans `ProductController` (classe créée si absente, `Requirement::DIGITS` appliqué sur `{id}`).

`/symfony:routing-define src/Controller/OrderController.php` — audit des routes du fichier (noms, méthodes bornées, requirements, duplications, ordre de priorité, usage de `redirectToRoute` ailleurs dans le code).

`/symfony:routing-define` sans argument — demande path, méthode, nom de route et comportement attendu.

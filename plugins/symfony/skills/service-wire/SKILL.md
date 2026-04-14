---
name: service-wire
description: Câble les arguments d'un service Symfony — scalaires, `%env(...)%`, `#[Autowire]`, bindings, alias, `#[Target]`. Déclenche sur "Cannot autowire", "injecter paramètre", "#[Autowire]", "%env()%", "alias service".
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /service-wire — Câbler les arguments d'un service

> **Utilise quand** l'autowiring échoue : argument scalaire, `%env()%`, choix d'implémentation, alias.
> **Pas quand** tu déclares un service neuf → `/symfony:service-define`.
> **Pas quand** tu collectes plusieurs services tagués → `/symfony:service-tags`.

Tu règles les cas où l'autowiring par type-hint ne suffit pas : argument scalaire, variable d'environnement, choix d'implémentation pour une interface, service nommé. Tu privilégies `#[Autowire]` sur l'argument (localisé, lisible) plutôt que d'éparpiller du YAML dans `services.yaml`.

## Détection préalable (obligatoire)

1. Lire `composer.json` — vérifier `symfony/framework-bundle`.
2. Lire `config/services.yaml` pour repérer `_defaults` et les éventuels `bind:` déjà en place.
3. Lire `config/packages/*.yaml` et `.env` / `.env.local` pour identifier les paramètres et env vars déjà déclarés.
4. Si le service n'est pas encore enregistré → passer par `/symfony:service-define` d'abord.

## Règles fondamentales

- **Attribut `#[Autowire]` par défaut** : configuration localisée sur l'argument, visible à la lecture de la classe. Ne basculer en YAML (`arguments:` ou `_defaults.bind`) que si plusieurs services partagent le même câblage ou si la classe n'est pas éditable (vendor).
- **Pas de `%parameter%` en dur dans le code** : un scalaire d'infra (URL, clé) vient de `parameters` (YAML) ou d'une env var, jamais d'un `const` dans la classe.
- **Env vars via `%env(...)%`** : jamais `$_ENV['FOO']` ni `getenv('FOO')` en direct dans un service. Symfony résout les env vars au moment de la compilation/runtime selon le processor (`string:`, `int:`, `bool:`, `json:`, `resolve:`, `default::`).
- **`_defaults.bind` pour partager un câblage** : quand plusieurs services dans le même fichier reçoivent le même `$adminEmail`, `$logger:@monolog.logger.request`, on le bind une fois globalement.
- **Un alias par implémentation par défaut** : si deux classes implémentent `PaymentGatewayInterface`, l'autowiring ne sait pas laquelle choisir. Déclarer un alias qui pointe vers l'implémentation par défaut, puis utiliser `#[Autowire(service: '...')]` ou `#[Target('...')]` pour les autres.
- **`$param` en camelCase côté PHP**, YAML utilise ce nom tel quel. `$adminEmail` → `$adminEmail:` dans le YAML, pas `$admin_email`.

## Arbre de décision : quel mécanisme pour quel besoin

| Besoin | Outil | Pourquoi |
|---|---|---|
| injecter un service concret dont le type est unique | **rien** — autowiring par type-hint | la règle par défaut |
| injecter un scalaire (email, URL, nombre) utilisé par **un seul** service | `#[Autowire(param: 'app.admin_email')]` | localisé, pas de YAML |
| injecter un env var | `#[Autowire(env: 'APP_API_KEY')]` | typé, processors disponibles |
| injecter une expression (dépend d'un autre service) | `#[Autowire(expression: 'service("foo").bar()')]` | evite un service intermédiaire |
| même scalaire pour 3+ services du fichier | `_defaults.bind.$varname` dans `services.yaml` | DRY |
| choisir parmi plusieurs implémentations d'une interface | **alias** + `#[Autowire(service: '...')]` ou `#[Target('nom')]` | désambiguïser |
| service vendor non autowirable | entrée YAML dédiée avec `arguments:` | pas d'attribut possible |

## Pattern 1 : injecter un paramètre scalaire

### Déclarer le paramètre

```yaml
# config/services.yaml
parameters:
    app.admin_email: 'manager@example.com'
    app.invoice.max_items: 500

services:
    _defaults:
        autowire: true
        autoconfigure: true
    App\:
        resource: '../src/'
        exclude: ['../src/{DependencyInjection,Entity,Kernel.php}']
```

### Injecter via `#[Autowire]` (préféré)

```php
declare(strict_types=1);

namespace App\Service;

use Symfony\Component\DependencyInjection\Attribute\Autowire;

final class InvoiceBuilder
{
    public function __construct(
        #[Autowire(param: 'app.admin_email')]
        private readonly string $adminEmail,
        #[Autowire(param: 'app.invoice.max_items')]
        private readonly int $maxItems,
    ) {}
}
```

L'autowiring injecte le scalaire au moment du build du container. Rien à ajouter dans `services.yaml` hors le bloc `parameters`.

### Alternative : YAML par service

Quand on ne peut pas modifier la classe (classe vendor, classe générée) :

```yaml
services:
    App\Service\InvoiceBuilder:
        arguments:
            $adminEmail: '%app.admin_email%'
            $maxItems: '%app.invoice.max_items%'
```

## Pattern 2 : injecter une variable d'environnement

### Déclarer dans `.env`

```bash
# .env
STRIPE_SECRET_KEY=sk_test_placeholder
BILLING_TIMEOUT=30
FEATURE_FLAGS={"beta":true,"dark":false}
```

### Injecter avec processor

```php
use Symfony\Component\DependencyInjection\Attribute\Autowire;

final class StripeClient
{
    public function __construct(
        #[Autowire(env: 'STRIPE_SECRET_KEY')]
        private readonly string $apiKey,

        #[Autowire(env: 'int:BILLING_TIMEOUT')]
        private readonly int $timeout,

        #[Autowire(env: 'json:FEATURE_FLAGS')]
        private readonly array $features,

        #[Autowire(env: 'default:prod:APP_ENV')]
        private readonly string $env,
    ) {}
}
```

Processors courants : `string:`, `int:`, `bool:`, `float:`, `json:`, `base64:`, `csv:`, `url:`, `query_string:`, `resolve:`, `default:<fallback>:`. Combinables : `env:int:default:30:BILLING_TIMEOUT`.

### Cas du secret en prod

`.env.local` n'est jamais commité (cf. `references/stacks/symfony.md` § Sécurité). En prod, la valeur vient des variables d'environnement du conteneur/Vault. Le processor `default:` permet d'avoir une valeur locale sans la mettre dans `.env` commité :

```php
#[Autowire(env: 'default::STRIPE_SECRET_KEY')]
private readonly string $apiKey,
```

Le `default::` (vide) déclenche une erreur au boot si l'env var est absente en prod — comportement souhaité pour un secret obligatoire.

## Pattern 3 : expression service

Quand un argument dépend du résultat d'un autre service et qu'on ne veut pas d'un service wrapper :

```php
use Symfony\Component\DependencyInjection\Attribute\Autowire;

final class FeatureFlag
{
    public function __construct(
        #[Autowire(expression: 'service("kernel").getEnvironment()')]
        private readonly string $env,
    ) {}
}
```

Syntaxe ExpressionLanguage. À éviter si la logique dépasse un appel simple — passer plutôt par un service dédié.

## Pattern 4 : bindings globaux (`_defaults.bind`)

Quand **plusieurs** services ont besoin du même scalaire ou du même service nommé, factoriser dans `_defaults.bind` :

```yaml
# config/services.yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true
        bind:
            $adminEmail: '%app.admin_email%'
            $projectDir: '%kernel.project_dir%'
            Psr\Log\LoggerInterface: '@monolog.logger.request'
            Psr\Log\LoggerInterface $auditLogger: '@monolog.logger.audit'
            iterable $paymentRules: !tagged_iterator app.payment_rule

    App\:
        resource: '../src/'
        exclude: ['../src/{DependencyInjection,Entity,Kernel.php}']
```

- `$varname: valeur` → match par nom d'argument.
- `TypeOrInterface: valeur` → match par type (override de l'autowiring standard).
- `TypeOrInterface $varname: valeur` → match combiné (deux loggers différents selon le nom d'argument).
- `!tagged_iterator <tag>` → injecte tous les services taggués (cf. `/symfony:service-tags`).

**Règle** : binder dans `_defaults.bind` seulement ce qui est partagé. Un scalaire utilisé par un unique service reste en `#[Autowire]` localisé.

## Pattern 5 : choisir parmi plusieurs implémentations

### Problème

```php
interface PaymentGatewayInterface { public function charge(int $cents): string; }

final class StripeGateway implements PaymentGatewayInterface { /* ... */ }
final class PaypalGateway implements PaymentGatewayInterface { /* ... */ }
```

L'autowiring d'un argument `PaymentGatewayInterface $gateway` échoue : `Cannot autowire service "...": argument $gateway of method __construct() references interface "PaymentGatewayInterface" but no such service exists. You should maybe alias this interface to one of these existing services: App\Payment\StripeGateway, App\Payment\PaypalGateway`.

### Solution 1 — alias par défaut + `#[Autowire(service:)]` pour les autres

```yaml
# config/services.yaml
services:
    # implémentation par défaut
    App\Payment\PaymentGatewayInterface: '@App\Payment\StripeGateway'
```

```php
use App\Payment\PaymentGatewayInterface;
use App\Payment\PaypalGateway;
use Symfony\Component\DependencyInjection\Attribute\Autowire;

final class CheckoutService
{
    public function __construct(
        private readonly PaymentGatewayInterface $gateway,               // Stripe par défaut
        #[Autowire(service: PaypalGateway::class)]
        private readonly PaymentGatewayInterface $paypal,                // Paypal explicite
    ) {}
}
```

### Solution 2 — `#[Target]` pour nommer les implémentations

Depuis Symfony 5.3, quand un argument suit la convention `$stripeGateway` ou `$paypalGateway`, l'autowiring les résout automatiquement sur les services correspondants. Pour forcer le match quand le nom diffère :

```php
use Symfony\Component\DependencyInjection\Attribute\Target;

final class CheckoutService
{
    public function __construct(
        #[Target('stripeGateway')]
        private readonly PaymentGatewayInterface $main,
        #[Target('paypalGateway')]
        private readonly PaymentGatewayInterface $fallback,
    ) {}
}
```

L'alias se déclare via `#[AsAlias]` côté implémentation :

```php
use Symfony\Component\DependencyInjection\Attribute\AsAlias;

#[AsAlias(id: PaymentGatewayInterface::class, name: 'stripeGateway')]
final class StripeGateway implements PaymentGatewayInterface {}
```

Choisir solution 1 si l'interface a une implémentation « par défaut » évidente, solution 2 si plusieurs alternatives coexistent sans hiérarchie.

## Pattern 6 : service vendor non autowirable

Quand une classe du vendor attend un scalaire ou un service sans type-hint :

```yaml
services:
    Acme\LegacyClient:
        arguments:
            $endpoint: '%env(ACME_ENDPOINT)%'
            $timeout: 30
            $httpClient: '@http_client'
            $cache: '@cache.app'
```

Pas d'échappatoire attribut puisqu'on ne modifie pas la classe.

## Pattern 7 : valeurs depuis une constante de classe

```yaml
services:
    App\Service\RateLimiter:
        arguments:
            $maxAttempts: !php/const App\Security\Throttle::MAX_ATTEMPTS
            $strategy: !php/enum App\Security\Strategy::Exponential
```

`!php/const` pour une constante, `!php/enum` pour une énumération. Utile pour garder la valeur unique côté PHP sans la dupliquer en YAML.

## Commandes utiles

```bash
# Qu'est-ce qui est autowirable pour ce type-hint ?
symfony console debug:autowiring PaymentGateway

# Comment ce service est-il câblé au final ?
symfony console debug:container App\\Service\\CheckoutService --show-arguments

# Résoudre les env vars pour voir ce qui sera injecté
symfony console debug:container --env-vars

# Valider la compilation (détecte bindings manquants, args non résolus)
symfony console lint:container
symfony console lint:container --resolve-env-vars
```

## Pièges fréquents

- **`#[Autowire(param:)]` vs `#[Autowire(value:)]`** : `param:` réfère à une clé de `parameters`, `value:` accepte un scalaire littéral. Mélanger les deux produit une `InvalidArgumentException`.
- **Env var redéclarée** : une env var mentionnée avec `#[Autowire(env:)]` mais absente de `.env` lève une erreur au build. Toujours la déclarer au minimum dans `.env` (valeur placeholder ou vide).
- **Bind trop large** : binder `string $adminEmail` dans `_defaults` capture tout `$adminEmail` du projet, y compris dans une classe où on ne le voulait pas. Si ambigüité, rester sur `#[Autowire]`.
- **`@` oublié en YAML** : `$logger: 'monolog.logger.request'` injecte la **chaîne**, pas le service. Il faut `'@monolog.logger.request'`.
- **Tests qui cassent après bump Symfony** : les processors d'env var et `#[Target]` évoluent. Toujours lancer `lint:container` dans la CI.

## Déroulement

### 1 — Cadrer

- Nature de l'argument : service concret, service interface ambigu, scalaire, env var, expression.
- Portée : un seul service (→ `#[Autowire]`) ou plusieurs (→ `_defaults.bind`).
- Modifiable ? Classe applicative (→ attribut) ou vendor (→ YAML).

### 2 — Appliquer

- Déclarer le scalaire / env var au bon endroit (`parameters`, `.env`).
- Poser l'attribut `#[Autowire]` sur l'argument, ou déclarer l'alias d'interface, ou compléter `services.yaml`.
- Ne pas dupliquer : si on ajoute un bind global, retirer les configurations individuelles qu'il remplace.

### 3 — Vérifier

```bash
symfony console debug:container App\\Service\\MonService --show-arguments
symfony console lint:container
vendor/bin/phpstan analyse src
```

Tester : un test unitaire qui instancie le service avec ses dépendances mockées valide la signature du constructeur. Un test functional qui récupère le service via `static::getContainer()->get(...)` valide le câblage réel.

### 4 — Clôture

Afficher :

- Argument(s) câblé(s) et mécanisme choisi.
- Fichier(s) modifié(s) : classe PHP, `services.yaml`, `parameters`, `.env`.
- Ce qui reste : tag custom (→ `/symfony:service-tags`), documentation de l'env var dans `.env` avec un commentaire si secret.

## Delta Sylius

- Beaucoup de services Sylius exposent des paramètres via `sylius_<bundle>.config`. Passer par la config Sylius plutôt que par `#[Autowire]` quand le paramètre est déjà exposé par le bundle.
- Les services `sylius.repository.*` sont publics et autowirables par alias (`OrderRepositoryInterface`). Pas besoin de binding manuel.

## Argument optionnel

`/symfony:service-wire src/Service/StripeClient.php` — audite les arguments du service et propose le câblage (attribut, bind, YAML).

`/symfony:service-wire "CheckoutService needs STRIPE_API_KEY env + default PaymentGatewayInterface → StripeGateway"` — applique le câblage décrit.

`/symfony:service-wire` sans argument — demande le service et l'argument à câbler.

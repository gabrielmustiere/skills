---
name: service-tags
description: Tags container Symfony — `#[AutoconfigureTag]`, `#[AsTaggedItem]`, `!tagged_iterator`, `!tagged_locator`, `#[AsDecorator]`, compiler pass. Déclenche sur "tagger service", "tagged_iterator", "AsDecorator", "strategy Symfony".
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /service-tags — Tags, collections, factories et décorateurs

> **Utilise quand** tu collectes une famille de services (strategy, chain) ou décores un service existant.
> **Pas quand** tu enregistres un service simple → `/symfony:service-define`.
> **Pas quand** tu câbles un argument scalaire/env → `/symfony:service-wire`.

Tu orchestres plusieurs services via le container : collection de règles métier, chain of responsibility, strategy pattern, décoration d'un service existant, construction via factory. Tu privilégies les attributs PHP (`#[AutoconfigureTag]`, `#[AsTaggedItem]`, `#[AsDecorator]`) à la configuration YAML, et tu ne passes aux Compiler Passes qu'en dernier recours.

## Détection préalable (obligatoire)

1. Lire `composer.json` — vérifier `symfony/framework-bundle`.
2. Lire `config/services.yaml` pour repérer `_defaults` (autowire/autoconfigure actifs attendus) et les éventuels bindings de `!tagged_iterator` déjà en place.
3. Si le service à tagger n'est pas encore enregistré → passer par `/symfony:service-define`.

## Règles fondamentales

- **Ne pas réinventer un pattern déjà taggué** : event listeners, commandes, messenger handlers, voters, twig extensions sont déjà auto-taggués par l'autoconfiguration (cf. `/symfony:service-define`). Tagger manuellement uniquement pour un **tag applicatif custom**.
- **`#[AutoconfigureTag]` sur l'interface** : le bon point d'ancrage pour « toute classe qui implémente `PaymentRuleInterface` est taggée ». Évite de tagger service par service.
- **Consommer via `iterable` ou `ServiceLocator`, jamais en scannant le container** : `!tagged_iterator` pour une chaîne à parcourir, `!tagged_locator` pour un accès par clé (lazy).
- **Priorité via `#[AsTaggedItem]`** : quand l'ordre compte (chain of responsibility), poser `priority` sur les classes. Plus grand = d'abord.
- **Décoration via `#[AsDecorator]` uniquement** : modifier un service vendor sans le remplacer. Jamais d'override de service vendor par redéfinition manuelle du même id.
- **Compiler Pass = dernier recours** : à utiliser quand aucun attribut ni YAML ne permet la manipulation souhaitée (ex : supprimer un service selon une condition dynamique au build). Sinon, c'est une over-engineering.

## Pattern 1 : collection de règles (iterator)

### Cas d'usage

Appliquer plusieurs règles de validation sur une commande, chacune décidant si elle valide ou non. Le consommateur les parcourt toutes.

### Définir l'interface et l'autoconfiguration

```php
// src/Order/Rule/OrderRuleInterface.php
declare(strict_types=1);

namespace App\Order\Rule;

use App\Order\Order;
use Symfony\Component\DependencyInjection\Attribute\AutoconfigureTag;

#[AutoconfigureTag('app.order_rule')]
interface OrderRuleInterface
{
    public function supports(Order $order): bool;
    public function validate(Order $order): void;
}
```

### Implémentations

```php
// src/Order/Rule/MinimumAmountRule.php
declare(strict_types=1);

namespace App\Order\Rule;

use App\Order\Order;
use Symfony\Component\DependencyInjection\Attribute\AsTaggedItem;

#[AsTaggedItem(priority: 100)]
final class MinimumAmountRule implements OrderRuleInterface
{
    public function supports(Order $order): bool
    {
        return true;
    }

    public function validate(Order $order): void
    {
        if ($order->totalCents() < 500) {
            throw new \DomainException('Minimum order is 5€');
        }
    }
}
```

Aucune ligne de config — `#[AutoconfigureTag]` tague l'implémentation, `#[AsTaggedItem]` donne sa priorité.

### Consommer la collection

```php
// src/Order/OrderValidator.php
declare(strict_types=1);

namespace App\Order;

use App\Order\Rule\OrderRuleInterface;
use Symfony\Component\DependencyInjection\Attribute\AutowireIterator;

final class OrderValidator
{
    public function __construct(
        /** @var iterable<OrderRuleInterface> */
        #[AutowireIterator('app.order_rule')]
        private readonly iterable $rules,
    ) {}

    public function validate(Order $order): void
    {
        foreach ($this->rules as $rule) {
            if ($rule->supports($order)) {
                $rule->validate($order);
            }
        }
    }
}
```

**Alternatives** :

- Via YAML binding global (utile si plusieurs services consomment la même collection) :

```yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true
        bind:
            iterable $orderRules: !tagged_iterator app.order_rule
```

- Via attribut sans binding : `#[AutowireIterator('app.order_rule')]` (depuis Symfony 6.3). Préféré pour un consommateur unique.

### Tri par priorité

L'ordre est celui déclaré dans le container sauf si une priorité est posée. Tri descendant (100 avant 50 avant 0). Pour trier par une autre clé :

```yaml
iterable $rules: !tagged_iterator { tag: app.order_rule, default_priority_method: getPriority }
```

`getPriority()` sur chaque classe retourne l'int. Pratique quand la priorité vient d'une config métier.

## Pattern 2 : sélection par clé (service locator)

### Cas d'usage

Choisir un handler parmi N en fonction d'une clé runtime (type d'événement, type de document, code pays). Le consommateur ne veut **pas** instancier tous les handlers, seulement celui demandé.

### Définir les handlers avec un index

```php
use Symfony\Component\DependencyInjection\Attribute\AsTaggedItem;

#[AsTaggedItem(index: 'invoice')]
final class InvoicePdfRenderer implements DocumentRendererInterface { /* ... */ }

#[AsTaggedItem(index: 'quote')]
final class QuotePdfRenderer implements DocumentRendererInterface { /* ... */ }

#[AsTaggedItem(index: 'receipt')]
final class ReceiptPdfRenderer implements DocumentRendererInterface { /* ... */ }
```

### Consommer via `ServiceLocator`

```php
use App\Document\DocumentRendererInterface;
use Psr\Container\ContainerInterface;
use Symfony\Component\DependencyInjection\Attribute\AutowireLocator;

final class DocumentDispatcher
{
    public function __construct(
        #[AutowireLocator('app.document_renderer')]
        private readonly ContainerInterface $renderers,
    ) {}

    public function render(string $type, Document $doc): string
    {
        if (!$this->renderers->has($type)) {
            throw new \InvalidArgumentException("No renderer for {$type}");
        }
        /** @var DocumentRendererInterface $renderer */
        $renderer = $this->renderers->get($type);

        return $renderer->render($doc);
    }
}
```

Le locator instancie le handler **uniquement** au `get()` — aucun coût pour les renderers non utilisés.

### Alternative via YAML

```yaml
services:
    App\Document\DocumentDispatcher:
        arguments:
            $renderers: !tagged_locator { tag: app.document_renderer, index_by: 'key' }
```

`index_by: 'key'` utilise l'attribut `key` du tag (ou `index` de `#[AsTaggedItem]`).

## Pattern 3 : factory

Utiliser une factory quand l'instanciation du service dépend d'une logique trop complexe pour un constructeur (choix selon env, appel à une API externe au boot, calcul à partir de plusieurs sources).

### Factory méthode statique

```php
// src/Http/HttpClientFactory.php
declare(strict_types=1);

namespace App\Http;

use Symfony\Component\HttpClient\HttpClient;
use Symfony\Contracts\HttpClient\HttpClientInterface;

final class HttpClientFactory
{
    public static function create(string $baseUrl, int $timeout): HttpClientInterface
    {
        return HttpClient::create([
            'base_uri' => $baseUrl,
            'timeout'  => $timeout,
            'headers'  => ['Accept' => 'application/json'],
        ]);
    }
}
```

```yaml
services:
    app.acme_http_client:
        class: Symfony\Contracts\HttpClient\HttpClientInterface
        factory: [App\Http\HttpClientFactory, create]
        arguments:
            $baseUrl: '%env(ACME_BASE_URL)%'
            $timeout: 10
```

### Factory service (méthode non-statique)

```yaml
services:
    App\Http\HttpClientFactory: ~

    app.acme_http_client:
        class: Symfony\Contracts\HttpClient\HttpClientInterface
        factory: ['@App\Http\HttpClientFactory', 'create']
        arguments:
            $baseUrl: '%env(ACME_BASE_URL)%'
            $timeout: 10
```

### `from_callable` (Symfony 6.3+)

```yaml
services:
    app.message_formatter:
        class: App\Service\MessageFormatterInterface
        from_callable: [!service { class: App\Service\MessageUtils }, format]
```

Crée un adaptateur qui résout un callable en implémentation d'une interface à une seule méthode.

## Pattern 4 : décoration (`#[AsDecorator]`)

### Cas d'usage

Ajouter un comportement autour d'un service existant (vendor, Symfony, Sylius) sans le remplacer : logging, cache, rate-limiting, transformation.

### Avec attribut (préféré)

```php
// src/Mailer/LoggingMailer.php
declare(strict_types=1);

namespace App\Mailer;

use Psr\Log\LoggerInterface;
use Symfony\Component\DependencyInjection\Attribute\AsDecorator;
use Symfony\Component\Mailer\MailerInterface;
use Symfony\Component\Mime\RawMessage;

#[AsDecorator(decorates: 'mailer.mailer')]
final class LoggingMailer implements MailerInterface
{
    public function __construct(
        private readonly MailerInterface $inner,
        private readonly LoggerInterface $logger,
    ) {}

    public function send(RawMessage $message, ?Envelope $envelope = null): void
    {
        $this->logger->info('Sending email', ['subject' => $message->getHeaders()->get('Subject')?->getBodyAsString()]);
        $this->inner->send($message, $envelope);
    }
}
```

- `decorates: 'mailer.mailer'` nomme le service cible (id).
- L'argument `$inner` reçoit automatiquement le service original — pas besoin de l'autowirer.
- Chaîner plusieurs décorateurs : `#[AsDecorator(decorates: 'mailer.mailer', priority: 10)]` (priorité haute = plus proche du service original).

### Avec YAML

```yaml
services:
    App\Mailer\LoggingMailer:
        decorates: 'mailer.mailer'
        arguments:
            $inner: '@.inner'
            $logger: '@logger'
```

Le placeholder `@.inner` (depuis 5.1) remplace l'ancien `@app.mailer.inner`.

### Quand décorer vs étendre

- **Décoration** : le comportement s'ajoute autour de toutes les occurrences d'utilisation du service existant (logging, cache, metrics). Transparent pour les consommateurs.
- **Alias vers une sous-classe** : le comportement remplace la logique pour un cas d'usage précis. Plus intrusif, moins composable.

En règle générale : si on veut **ajouter**, décorer. Si on veut **remplacer**, alias.

## Pattern 5 : `#[AsAlias]` pour exposer un service sous un autre nom

```php
use Symfony\Component\DependencyInjection\Attribute\AsAlias;

#[AsAlias(id: PaymentGatewayInterface::class)]
final class StripeGateway implements PaymentGatewayInterface { /* ... */ }
```

Équivalent attribut du `App\Payment\PaymentGatewayInterface: '@App\Payment\StripeGateway'` en YAML. Utile pour désigner l'implémentation par défaut d'une interface (complémentaire de `/symfony:service-wire` pour le câblage).

## Pattern 6 : Compiler Pass (dernier recours)

Un Compiler Pass manipule le container **pendant la compilation**, après que toutes les extensions bundles ont tourné. À utiliser pour :

- Supprimer dynamiquement un service selon une condition calculée (pas un simple `when@env`).
- Injecter dans tous les services d'un tag une dépendance conditionnelle.
- Valider que certains services existent / sont bien taggués.

### Exemple : injecter un logger dans tous les services taggués

```php
// src/DependencyInjection/Compiler/AddAuditLoggerPass.php
declare(strict_types=1);

namespace App\DependencyInjection\Compiler;

use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Reference;

final class AddAuditLoggerPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container): void
    {
        if (!$container->has('monolog.logger.audit')) {
            return;
        }

        foreach ($container->findTaggedServiceIds('app.audited_action') as $id => $_) {
            $container->getDefinition($id)->addMethodCall(
                'setAuditLogger',
                [new Reference('monolog.logger.audit')],
            );
        }
    }
}
```

```php
// src/Kernel.php
protected function build(ContainerBuilder $container): void
{
    $container->addCompilerPass(new AddAuditLoggerPass());
}
```

**Avant d'écrire un Compiler Pass** : se demander si l'une de ces alternatives suffit.

1. `#[AutoconfigureTag]` + `!tagged_iterator` pour collecter.
2. `#[AsDecorator]` pour wrapper.
3. `_defaults.bind` pour injecter partout.
4. `when@env` ou `#[When]` pour conditionner par environnement.

Dans 95 % des projets applicatifs, un Compiler Pass n'est pas nécessaire. On en rencontre surtout dans les bundles tiers.

## Pattern 7 : `tagged_iterator` avec exclusion

```yaml
iterable $rules: !tagged_iterator { tag: app.rule, exclude: ['app.rule.disabled'] }
```

Utile pour exclure un sous-ensemble (ex : règle désactivée par feature flag, appliquée via un second tag).

## Commandes de debug

```bash
# Tous les services taggués avec un tag
symfony console debug:container --tag=app.order_rule

# Tous les tags déclarés
symfony console debug:container --tags

# Détail d'un service décoré (voir la chaîne)
symfony console debug:container mailer.mailer

# Valider la compilation
symfony console lint:container
```

## Pièges fréquents

- **Tag sans autoconfiguration ni config manuelle** : un service n'est pas taggué automatiquement par le simple fait d'implémenter une interface — il faut `#[AutoconfigureTag]` sur l'interface **et** `autoconfigure: true` dans `_defaults` (qui est le défaut Flex, donc en général OK).
- **Ordre non déterministe** : sans `priority`, deux implémentations avec la même priorité (0 par défaut) sortent dans l'ordre de déclaration du container, qui peut varier. Poser explicitement une priorité si l'ordre importe.
- **`iterable` vs `array`** : consommer en `iterable` permet au container d'instancier lazy. En forçant `array`, on perd la lazy-instantiation. Préférer `iterable` partout.
- **Décorateur marqué `final`** : OK. Le service décoré (cible) n'a pas besoin d'être modifiable — Symfony remplace l'id, il ne fait pas d'héritage.
- **Compiler Pass qui mute le container sans priorité** : les passes ont un ordre (`PassConfig::TYPE_BEFORE_OPTIMIZATION` par défaut). Si un pass dépend d'un autre, passer `$passType` et `$priority` explicitement à `addCompilerPass()`.
- **`ServiceLocator` utilisé pour éviter l'autowiring** : si les clés sont connues à l'avance, autowirer directement les services est plus clair. Le locator brille seulement pour un dispatch runtime.

## Déroulement

### 1 — Cadrer

- Collection (`iterable`), sélection par clé (`ServiceLocator`), wrapper (décoration), ou construction custom (factory) ?
- L'interface à tagger existe-t-elle ? Sinon, la créer (cf. `/symfony:service-define` pour la classe).
- Priorité / index nécessaire ?

### 2 — Implémenter

- `#[AutoconfigureTag('app.<nom>')]` sur l'interface si collection.
- `#[AsTaggedItem(priority: N, index: 'key')]` sur chaque implémentation si nécessaire.
- Consommateur : `#[AutowireIterator]` / `#[AutowireLocator]` ou binding YAML.
- Décoration : `#[AsDecorator]` sur la classe wrapper.
- Factory : entrée YAML avec `factory:` ou `from_callable:`.

### 3 — Vérifier

```bash
symfony console debug:container --tag=app.order_rule
symfony console lint:container
vendor/bin/phpstan analyse src
```

Test unitaire du consommateur en passant un `new \ArrayIterator([$rule1, $rule2])` en argument, puis test functional qui récupère le consommateur via `static::getContainer()->get(...)` pour valider que le tag collecte bien tous les services.

### 4 — Clôture

Afficher :

- Tag utilisé (`app.<nom>`).
- Services taggués (résultat de `debug:container --tag=...`).
- Consommateur et mécanisme (`iterable`, `ServiceLocator`, décoration, factory).
- Ce qui reste : tests, documentation du tag si public (bundle partagé).

## Delta Sylius

- Sylius expose de nombreux tags applicatifs (`sylius.order_processor`, `sylius.tax_calculator`, `sylius.shipping_method_resolver`). Utiliser `#[AsDecorator]` pour wrapper, ou `#[AsTaggedItem]` pour ajouter une implémentation au tag existant.
- Les extensions de checkout passent par `sylius.order_processor` (chaîne de processors ordonnée par priorité). Ajouter un processor custom = nouvelle classe taggée `sylius.order_processor` avec `priority:` réfléchie par rapport aux processors vendor.
- Pour des factories Sylius : utiliser les abstractions `Sylius\Component\Resource\Factory\FactoryInterface` plutôt que d'écrire une factory Symfony brute.

## Argument optionnel

`/symfony:service-tags OrderRuleInterface app.order_rule` — monte la collection pour l'interface nommée sous le tag donné.

`/symfony:service-tags "décorer mailer.mailer avec LoggingMailer"` — crée le décorateur.

`/symfony:service-tags src/Order/OrderValidator.php` — audit : détecte si le service devrait consommer une collection taggée plutôt que d'injecter des implémentations en dur.

`/symfony:service-tags` sans argument — demande le pattern cible (collection, locator, décoration, factory).

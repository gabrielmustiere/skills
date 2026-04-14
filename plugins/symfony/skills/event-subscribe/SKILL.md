---
name: event-subscribe
description: Crée un Event Subscriber Symfony — `EventSubscriberInterface`, `getSubscribedEvents()`, multi-callbacks, priorités. Déclenche sur "event subscriber", "getSubscribedEvents", "KernelEvents", "before/after filter".
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /event-subscribe — Event Subscriber Symfony

> **Utilise quand** une classe orchestre plusieurs callbacks liés (before/after filter, multi-événements).
> **Pas quand** un seul callback suffit → `/symfony:event-listen`.
> **Pas quand** tu crées/dispatches l'événement → `/symfony:event-dispatch`.

Tu crées une classe qui centralise la réaction à un ou plusieurs événements du kernel (ou custom) en implémentant `EventSubscriberInterface`. Le subscriber déclare lui-même ses abonnements via `getSubscribedEvents()` — c'est le bon choix quand la logique couvre plusieurs événements liés ou quand on veut que la classe documente elle-même ce à quoi elle répond.

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine — vérifier `symfony/event-dispatcher` (généralement tiré par `symfony/framework-bundle`).
   - Absent → afficher : *« Ce skill cible Symfony, je ne trouve pas `symfony/event-dispatcher`. On continue quand même ou on change d'approche ? »* et attendre.
2. Lire `config/services.yaml` pour vérifier `_defaults.autoconfigure: true` (indispensable : c'est l'autoconfiguration qui tague `kernel.event_subscriber` sans config).
3. Si l'événement visé est un événement **custom**, vérifier qu'il est dispatché quelque part (`grep -r "dispatch(" src/`). Sinon renvoyer vers `/symfony:event-dispatch`.

## Règles fondamentales

- **Subscriber vs Listener** :
  - Subscriber = classe qui **déclare elle-même** ses événements via `getSubscribedEvents()`. Préféré quand la classe réagit à plusieurs événements liés ou qu'on veut auto-documenter les abonnements.
  - Listener = classe taggée par configuration (YAML ou `#[AsEventListener]`). Préféré pour un écouteur mono-événement, pour un code qu'on veut activer/désactiver par config, ou dans un bundle partagé. Voir `/symfony:event-listen`.
- **Nom d'événement = FQCN** : toujours utiliser la classe de l'événement (`RequestEvent::class`) plutôt que la constante string (`'kernel.request'`). Type-safe, refactorable, explicite. Les constantes `KernelEvents::REQUEST` sont acceptables comme alias lisible.
- **Priorité = int** : plus grand = exécuté plus tôt. Défaut `0`. Les écouteurs internes Symfony vivent dans `[-256, 256]`. Pour s'insérer, lire `debug:event-dispatcher <event>` et choisir une priorité entre les voisins.
- **Une méthode courte par abonnement** : la méthode callback reçoit l'événement, extrait ce dont elle a besoin, délègue le vrai travail à un service injecté. On ne met pas de logique métier lourde dans la méthode elle-même.
- **`declare(strict_types=1)`** en tête, classe `final`, constructeur typehinté (injection standard Symfony — l'autoconfigure tague en `kernel.event_subscriber` même sur une classe `final`).
- **Pas de `static` en dehors de `getSubscribedEvents()`** : cette méthode est statique par contrat. Le reste de la classe est normal.
- **Ne pas stocker d'état entre événements** : un subscriber est un service, donc singleton. Si un événement doit passer une info à un autre, passer par l'objet `Event` lui-même ou `Request::$attributes`.

## Structure minimale

```php
// src/EventSubscriber/OrderCreatedSubscriber.php
declare(strict_types=1);

namespace App\EventSubscriber;

use App\Event\OrderCreatedEvent;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

final class OrderCreatedSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            OrderCreatedEvent::class => 'onOrderCreated',
        ];
    }

    public function onOrderCreated(OrderCreatedEvent $event): void
    {
        // réaction courte, délégation à un service si besoin
    }
}
```

Placer sous `src/EventSubscriber/` (convention Symfony Flex — le resource `App\:` couvre le dossier, l'autoconfigure pose le tag `kernel.event_subscriber`). Aucune ligne de config YAML.

## Formes de retour de `getSubscribedEvents()`

Trois syntaxes acceptées, toutes valides :

```php
public static function getSubscribedEvents(): array
{
    return [
        // 1) Un callback sans priorité
        OrderCreatedEvent::class => 'onOrderCreated',

        // 2) Un callback avec priorité (int)
        ProductUpdatedEvent::class => ['onProductUpdated', 100],

        // 3) Plusieurs callbacks sur le même événement, chacun avec sa priorité
        ExceptionEvent::class => [
            ['logException',    10],
            ['transformToJson',  0],
            ['notifyOps',      -10],
        ],
    ];
}
```

Quand l'ordre importe entre callbacks internes (ex : logger **avant** transformer pour capturer l'exception brute), passer par la forme 3. Plus grand = plus tôt.

## Pattern 1 : before/after filter via kernel events

Cas d'usage typique : valider un token d'auth sur certaines routes (`kernel.controller`) puis signer la réponse (`kernel.response`). Les deux réactions partagent l'état via `Request::$attributes`.

```php
// src/EventSubscriber/TokenSubscriber.php
declare(strict_types=1);

namespace App\EventSubscriber;

use App\Controller\TokenAuthenticatedController;
use Symfony\Component\DependencyInjection\Attribute\Autowire;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\ControllerEvent;
use Symfony\Component\HttpKernel\Event\ResponseEvent;
use Symfony\Component\HttpKernel\Exception\AccessDeniedHttpException;
use Symfony\Component\HttpKernel\KernelEvents;

final class TokenSubscriber implements EventSubscriberInterface
{
    public function __construct(
        #[Autowire(param: 'app.api_tokens')]
        private readonly array $tokens,
    ) {}

    public function onKernelController(ControllerEvent $event): void
    {
        $controller = $event->getController();
        if (is_array($controller)) {
            $controller = $controller[0];
        }

        if (!$controller instanceof TokenAuthenticatedController) {
            return;
        }

        $token = $event->getRequest()->query->get('token');
        if (!in_array($token, $this->tokens, true)) {
            throw new AccessDeniedHttpException('Invalid token');
        }

        $event->getRequest()->attributes->set('auth_token', $token);
    }

    public function onKernelResponse(ResponseEvent $event): void
    {
        $token = $event->getRequest()->attributes->get('auth_token');
        if (null === $token) {
            return;
        }

        $response = $event->getResponse();
        $response->headers->set(
            'X-Content-Hash',
            sha1($response->getContent().$token),
        );
    }

    public static function getSubscribedEvents(): array
    {
        return [
            KernelEvents::CONTROLLER => 'onKernelController',
            KernelEvents::RESPONSE   => 'onKernelResponse',
        ];
    }
}
```

Points clés :

- `ControllerEvent` et `ResponseEvent` sont type-hintés sur les méthodes callbacks — leur typage est la source de vérité.
- L'état (`auth_token`) transite via `Request::$attributes`, pas via une propriété d'instance.
- L'interface marqueur `TokenAuthenticatedController` est portée par les contrôleurs concernés — le subscriber ne s'active que pour eux.
- `#[Autowire(param: 'app.api_tokens')]` injecte un paramètre container plutôt qu'un service (cf. `/symfony:service-wire`).

## Pattern 2 : plusieurs callbacks ordonnés sur un même événement

Cas d'usage : gérer les exceptions en plusieurs étapes distinctes (log brut, transformation en JSON, notification ops) qui doivent s'exécuter dans un ordre précis.

```php
// src/EventSubscriber/ExceptionHandlingSubscriber.php
declare(strict_types=1);

namespace App\EventSubscriber;

use App\Notification\OpsNotifier;
use Psr\Log\LoggerInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpKernel\Event\ExceptionEvent;

final class ExceptionHandlingSubscriber implements EventSubscriberInterface
{
    public function __construct(
        private readonly LoggerInterface $logger,
        private readonly OpsNotifier $ops,
    ) {}

    public static function getSubscribedEvents(): array
    {
        return [
            ExceptionEvent::class => [
                ['logException',     100],
                ['transformToJson',    0],
                ['notifyOps',       -100],
            ],
        ];
    }

    public function logException(ExceptionEvent $event): void
    {
        $this->logger->error(
            $event->getThrowable()->getMessage(),
            ['exception' => $event->getThrowable()],
        );
    }

    public function transformToJson(ExceptionEvent $event): void
    {
        $throwable = $event->getThrowable();
        $event->setResponse(new JsonResponse(
            ['error' => $throwable->getMessage()],
            500,
        ));
    }

    public function notifyOps(ExceptionEvent $event): void
    {
        // tourne après transformToJson ; on notifie uniquement si toujours 5xx
        $response = $event->getResponse();
        if ($response !== null && $response->getStatusCode() >= 500) {
            $this->ops->notify($event->getThrowable());
        }
    }
}
```

Logique : `logException` (priorité 100) capture d'abord l'exception brute, `transformToJson` (0) produit la réponse, `notifyOps` (-100) exploite la réponse désormais disponible.

## Pattern 3 : subscriber sur un événement custom

Cas d'usage : un mailer applicatif dispatch un `AfterMailSentEvent`, un subscriber archive chaque envoi. Voir `/symfony:event-dispatch` pour la création de l'événement et le dispatch.

```php
// src/EventSubscriber/ArchiveSentMailSubscriber.php
declare(strict_types=1);

namespace App\EventSubscriber;

use App\Event\AfterMailSentEvent;
use App\Mail\MailArchive;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

final class ArchiveSentMailSubscriber implements EventSubscriberInterface
{
    public function __construct(
        private readonly MailArchive $archive,
    ) {}

    public static function getSubscribedEvents(): array
    {
        return [
            AfterMailSentEvent::class => 'onAfterMailSent',
        ];
    }

    public function onAfterMailSent(AfterMailSentEvent $event): void
    {
        $this->archive->store(
            $event->getMessageId(),
            $event->getRecipient(),
            $event->getSubject(),
        );
    }
}
```

Aucune config : l'autoconfigure tague, le dispatcher appelle.

## Événements du kernel les plus utiles

| Constante / classe | Moment | Usage typique |
|---|---|---|
| `KernelEvents::REQUEST` / `RequestEvent` | Avant résolution du contrôleur | Routing applicatif, courts-circuit (`setResponse`) |
| `KernelEvents::CONTROLLER` / `ControllerEvent` | Contrôleur résolu, avant appel | Vérifs sur le contrôleur, enrichir la requête |
| `KernelEvents::CONTROLLER_ARGUMENTS` / `ControllerArgumentsEvent` | Arguments résolus, avant appel | Modifier les arguments passés au contrôleur |
| `KernelEvents::VIEW` / `ViewEvent` | Contrôleur a renvoyé autre chose qu'une `Response` | Transformer en `Response` (ex : sérialisation auto) |
| `KernelEvents::RESPONSE` / `ResponseEvent` | Juste avant l'envoi de la réponse | Headers, cookies, CSP, signing |
| `KernelEvents::FINISH_REQUEST` / `FinishRequestEvent` | Après envoi de la réponse principale | Cleanup, reset de locale |
| `KernelEvents::TERMINATE` / `TerminateEvent` | Après réponse envoyée au client | Tâches lourdes non bloquantes |
| `KernelEvents::EXCEPTION` / `ExceptionEvent` | Exception non capturée | Transformation en `Response` |
| `ConsoleEvents::COMMAND`, `TERMINATE`, `ERROR` | CLI | Instrumentation des commandes |

## Vérifier l'abonnement

```bash
# tous les subscribers + listeners, ordre d'exécution
symfony console debug:event-dispatcher

# un événement spécifique (FQCN ou string)
symfony console debug:event-dispatcher "Symfony\Component\HttpKernel\Event\ExceptionEvent"
symfony console debug:event-dispatcher kernel.exception

# matching partiel
symfony console debug:event-dispatcher kernel

# dispatcher non-global (ex : firewall security)
symfony console debug:event-dispatcher --dispatcher=security.event_dispatcher.main
```

La sortie montre chaque callback avec sa priorité et son service d'origine — la référence pour vérifier que l'abonnement est pris en compte et inséré au bon endroit.

## Tests

### Test unitaire (pur PHP, sans container)

```php
declare(strict_types=1);

namespace App\Tests\EventSubscriber;

use App\Event\AfterMailSentEvent;
use App\EventSubscriber\ArchiveSentMailSubscriber;
use App\Mail\MailArchive;
use PHPUnit\Framework\TestCase;

final class ArchiveSentMailSubscriberTest extends TestCase
{
    public function test_it_archives_sent_mail(): void
    {
        $archive = $this->createMock(MailArchive::class);
        $archive->expects(self::once())
            ->method('store')
            ->with('msg-42', 'alice@example.com', 'Welcome');

        $subscriber = new ArchiveSentMailSubscriber($archive);
        $subscriber->onAfterMailSent(
            new AfterMailSentEvent('msg-42', 'alice@example.com', 'Welcome'),
        );
    }

    public function test_it_subscribes_to_after_mail_sent(): void
    {
        self::assertArrayHasKey(
            AfterMailSentEvent::class,
            ArchiveSentMailSubscriber::getSubscribedEvents(),
        );
    }
}
```

### Test d'intégration via `EventDispatcher`

```php
$dispatcher = new \Symfony\Component\EventDispatcher\EventDispatcher();
$dispatcher->addSubscriber(new ArchiveSentMailSubscriber($archive));

$dispatcher->dispatch(new AfterMailSentEvent('msg-42', 'alice@example.com', 'Welcome'));
```

Plus fidèle si on veut aussi tester l'ordre entre plusieurs subscribers.

## Pièges fréquents

- **Constante `KernelEvents::RESPONSE` vs classe `ResponseEvent`** : les deux fonctionnent. Préférer `FQCN::class` pour la cohérence, utiliser la constante uniquement si elle rend l'intention plus lisible (ex : `KernelEvents::TERMINATE` sans classe dédiée attendue).
- **Subscriber non taggué** : si `autoconfigure: false` quelque part ou si la classe est dans un dossier exclu (`src/Kernel.php`, `src/Entity/`), le tag manque. Vérifier avec `debug:container App\\EventSubscriber\\NomSubscriber` → champ `Tags` doit contenir `kernel.event_subscriber`.
- **Exception dans un subscriber `kernel.response`** : remonte et casse la réponse. Toujours encadrer les effets de bord risqués (appel HTTP externe, I/O) par un try/catch qui log sans propager, ou déplacer vers `KernelEvents::TERMINATE`.
- **Priorité choisie à l'aveugle** : les écouteurs internes Symfony (firewall, routing, profiler) utilisent des priorités précises. Avant d'insérer avec `priority: 10`, lancer `debug:event-dispatcher <event>` pour voir où on tombe.
- **Subscriber qui mute l'événement puis appelle `stopPropagation()`** : légitime mais à documenter — les subscribers suivants ne verront jamais l'événement. Rare. Préférer laisser la chaîne continuer.
- **Même subscriber abonné au même événement deux fois** dans `getSubscribedEvents()` : Symfony silencieusement exécute les deux callbacks. Erreur de copier-coller typique — vérifier après refactor.

## Déroulement

### 1 — Cadrer

- Combien d'événements ? Un seul → candidate pour `/symfony:event-listen` (plus léger). Plusieurs liés → subscriber.
- Événement kernel existant ou custom ? Custom → s'assurer qu'il est dispatché (`/symfony:event-dispatch`).
- Ordre d'exécution contraint ? Fixer les priorités dès le départ.

### 2 — Implémenter

- Classe sous `src/EventSubscriber/<Nom>Subscriber.php`, `final`, `declare(strict_types=1)`.
- `implements EventSubscriberInterface`, `getSubscribedEvents(): array` statique.
- Une méthode callback par abonnement, typehintée sur la classe d'événement.
- Dépendances injectées par constructeur (jamais d'état mutable entre appels).

### 3 — Vérifier

```bash
symfony console debug:event-dispatcher <Event>::class
symfony console debug:container App\\EventSubscriber\\NomSubscriber
symfony console lint:container
vendor/bin/phpstan analyse src
```

Test unitaire de chaque callback + test que `getSubscribedEvents()` renvoie bien la map attendue.

### 4 — Clôture

Afficher :

- Fichier créé + dossier.
- Événements abonnés (sortie `debug:event-dispatcher`).
- Priorité(s) choisie(s) et justification (voisins dans la chaîne).
- Ce qui reste : tests, éventuellement `/symfony:event-dispatch` si événement custom à créer, `/symfony:event-listen` si on veut finalement un écouteur mono-événement.

## Delta Sylius

- Sylius dispatche **beaucoup** d'événements génériques (`GenericEvent`) autour des resources (`sylius.product.pre_create`, `sylius.order.post_update`, etc.). Pour s'y abonner, la signature du callback reçoit `Symfony\Component\EventDispatcher\GenericEvent` et on récupère la resource via `$event->getSubject()`.

```php
use Symfony\Component\EventDispatcher\GenericEvent;

public function onProductPostCreate(GenericEvent $event): void
{
    /** @var \Sylius\Component\Core\Model\ProductInterface $product */
    $product = $event->getSubject();
    // ...
}

public static function getSubscribedEvents(): array
{
    return [
        'sylius.product.post_create' => 'onProductPostCreate',
    ];
}
```

- Sylius utilise toujours les noms string (`sylius.<resource>.<hook>`), pas un FQCN. C'est l'exception à la règle FQCN — imposée par le bus Sylius.
- Les hooks disponibles (`pre_create`, `post_create`, `initialize_*`, `pre_update`, `post_update`, `pre_delete`, `post_delete`) viennent de `Sylius\Bundle\ResourceBundle\Controller\ResourceController`. Consulter `grep -rn "eventDispatcher->dispatch" vendor/sylius/resource-bundle/` pour la liste exhaustive.

## Argument optionnel

`/symfony:event-subscribe OrderCreatedEvent` — crée `OrderCreatedSubscriber` abonné à l'événement nommé, avec un callback stub.

`/symfony:event-subscribe "kernel.response + kernel.controller pour signer les réponses API"` — monte un subscriber multi-événements (pattern before/after filter).

`/symfony:event-subscribe src/EventSubscriber/FooSubscriber.php` — audit d'un subscriber existant (interface implémentée, `getSubscribedEvents()` cohérent, priorités documentées, callbacks courts).

`/symfony:event-subscribe` sans argument — demande l'événement (ou la liste) cible et la logique métier.

---
name: event-listen
description: Crée un Event Listener Symfony — `#[AsEventListener]` ou tag `kernel.event_listener`, `__invoke()`, priorité. Déclenche sur "event listener", "AsEventListener", "écouter événement", "onKernelRequest", "onKernelException".
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /event-listen — Event Listener Symfony

> **Utilise quand** une seule méthode réagit à un événement (kernel, security, custom).
> **Pas quand** plusieurs événements liés dans la même classe → `/symfony:event-subscribe`.
> **Pas quand** tu crées/dispatches l'événement → `/symfony:event-dispatch`.

Tu attaches un callback à un événement sans passer par `EventSubscriberInterface`. Tu utilises l'attribut `#[AsEventListener]` (PHP 8+, préféré) ou le tag YAML `kernel.event_listener` pour les cas où la config externe est nécessaire.

## Détection préalable (obligatoire)

1. Lire `composer.json` — vérifier `symfony/event-dispatcher` (tiré par `symfony/framework-bundle`).
2. Lire `config/services.yaml` — vérifier `_defaults.autoconfigure: true` (indispensable pour que `#[AsEventListener]` soit honoré sans config manuelle).
3. Si plusieurs événements dans la même classe **et** que la logique est liée → suggérer `/symfony:event-subscribe` avant d'aller plus loin. `AsEventListener` multi-événements reste OK, mais le subscriber auto-documente mieux la liste des abonnements.
4. Événement custom cible → vérifier qu'il est dispatché (`grep -r "->dispatch(" src/`). Sinon `/symfony:event-dispatch`.

## Règles fondamentales

- **Attribut > YAML** : `#[AsEventListener]` est la forme canonique depuis Symfony 6.1. Le tag YAML n'est nécessaire que si le listener est une classe qu'on ne peut pas modifier (vendor) ou si on veut désactiver l'abonnement par environnement via `when@env`.
- **`__invoke()` pour le cas single-event/single-method** : si le listener n'a qu'un callback, la méthode magique `__invoke()` est la plus lisible — plus besoin de nommer la méthode, l'attribut sur la classe suffit.
- **Inférence par type-hint** : posé sur une méthode publique, `#[AsEventListener]` sans `event:` infère le type d'événement depuis le type-hint du paramètre. Préféré quand possible (moins de duplication).
- **Priorité** : `priority` en int. Plus grand = plus tôt. Défaut `0`. Rangée interne Symfony `[-256, 256]`. Lire `debug:event-dispatcher <event>` avant de choisir pour éviter d'écraser/masquer un listener du framework.
- **Dispatcher nommé** : par défaut, tout listener s'attache au dispatcher global. Pour un dispatcher dédié (firewall security, bundle tiers), passer `dispatcher: 'security.event_dispatcher.main'` sur l'attribut.
- **`declare(strict_types=1)`**, classe `final`, injection constructeur — mêmes règles que partout.
- **Pas d'état interne** : un listener est un service singleton. Pas de propriété mutable qui dépendrait d'un événement pour être lue par un autre.

## Forme 1 : attribut sur la classe avec `__invoke()`

Le cas le plus léger — un callback, un événement.

```php
// src/EventListener/ExceptionListener.php
declare(strict_types=1);

namespace App\EventListener;

use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Event\ExceptionEvent;
use Symfony\Component\HttpKernel\Exception\HttpExceptionInterface;

#[AsEventListener]
final class ExceptionListener
{
    public function __invoke(ExceptionEvent $event): void
    {
        $throwable = $event->getThrowable();

        $response = new Response(
            sprintf('Error: %s (code %d)', $throwable->getMessage(), $throwable->getCode()),
        );
        $response->headers->set('Content-Type', 'text/plain; charset=utf-8');

        if ($throwable instanceof HttpExceptionInterface) {
            $response->setStatusCode($throwable->getStatusCode());
            $response->headers->replace($throwable->getHeaders());
        } else {
            $response->setStatusCode(Response::HTTP_INTERNAL_SERVER_ERROR);
        }

        $event->setResponse($response);
    }
}
```

L'attribut sans argument fonctionne parce que :

1. La classe est un service (via resource `App\:`),
2. `autoconfigure` pose le tag `kernel.event_listener`,
3. Symfony tente `__invoke()` et lit le type-hint `ExceptionEvent` → abonnement automatique.

## Forme 2 : attribut sur la classe, méthode explicite

Quand la classe n'a pas `__invoke()` (nom de méthode plus parlant, ou plusieurs méthodes utilitaires).

```php
#[AsEventListener(event: ExceptionEvent::class, method: 'onKernelException')]
final class ExceptionListener
{
    public function onKernelException(ExceptionEvent $event): void
    {
        // ...
    }
}
```

Sans `method`, Symfony essaie dans l'ordre :

1. `__invoke()`,
2. `on<PascalCaseEventName>()` — ex : `onKernelException`, `onKernelRequest`,
3. erreur.

Poser `method` explicite évite la résolution par convention et rend le wiring lisible au premier coup d'œil.

## Forme 3 : attribut sur la méthode

Permet un listener multi-événements dans une seule classe, tout en gardant la déclaration locale à chaque méthode.

```php
// src/EventListener/MyMultiListener.php
declare(strict_types=1);

namespace App\EventListener;

use App\Event\CustomEvent;
use App\Event\AnotherEvent;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

final class MyMultiListener
{
    #[AsEventListener]
    public function onCustomEvent(CustomEvent $event): void
    {
        // type inféré via le type-hint
    }

    #[AsEventListener(event: 'foo', priority: 42)]
    public function onFoo(): void
    {
        // événement nommé par string (pas de classe dédiée côté dispatcher)
    }

    #[AsEventListener]
    public function onUnionEvent(CustomEvent|AnotherEvent $event): void
    {
        // union type : abonnement aux deux événements
    }
}
```

Règle : l'attribut sans `event:` ne fonctionne que si le type-hint est présent et qu'il pointe vers une classe d'événement (ou une union de classes). Pour un nom d'événement string custom (`'foo'`, `'mailer.pre_send'`), passer `event:` explicitement.

## Forme 4 : tag YAML (cas résiduels)

À réserver aux listeners vendor qu'on ne peut pas modifier, ou aux abonnements conditionnels par environnement.

```yaml
# config/services.yaml
services:
    App\EventListener\ExceptionListener:
        tags:
            - name: kernel.event_listener
              event: kernel.exception
              method: onKernelException
              priority: 10
```

Syntaxe courte équivalente quand on suit les conventions de nommage :

```yaml
services:
    App\EventListener\ExceptionListener:
        tags:
            - { name: kernel.event_listener, event: kernel.exception }
```

Résolution de la méthode par Symfony (identique à l'attribut) :

1. `method:` explicite,
2. `__invoke()`,
3. `on<PascalCaseEventName>()`,
4. erreur.

### Abonnement par environnement

```yaml
when@dev:
    services:
        App\EventListener\ProfilerListener:
            tags:
                - { name: kernel.event_listener, event: kernel.response }
```

Le listener n'est enregistré qu'en `dev`. Alternative : attribut `#[When(env: 'dev')]` sur la classe pour qu'elle ne soit plus un service hors dev — plus propre quand on maîtrise la classe.

## Pattern 1 : listener `kernel.request` pour un header global

```php
// src/EventListener/EnforceJsonAcceptListener.php
declare(strict_types=1);

namespace App\EventListener;

use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\HttpKernel\Event\RequestEvent;

#[AsEventListener(priority: 32)]
final class EnforceJsonAcceptListener
{
    public function __invoke(RequestEvent $event): void
    {
        $request = $event->getRequest();
        if (!str_starts_with($request->getPathInfo(), '/api/')) {
            return;
        }

        if (!$request->headers->has('Accept')) {
            $request->headers->set('Accept', 'application/json');
        }
    }
}
```

Priorité 32 : après le `RouterListener` interne (priorité 32 aussi ? vérifier `debug:event-dispatcher kernel.request` pour caler) — la règle est de toujours valider l'insertion avec la commande debug plutôt que de deviner.

## Pattern 2 : listener sur dispatcher dédié (firewall security)

```php
// src/EventListener/LoginSuccessListener.php
declare(strict_types=1);

namespace App\EventListener;

use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\Security\Http\Event\LoginSuccessEvent;

#[AsEventListener(
    event: LoginSuccessEvent::class,
    dispatcher: 'security.event_dispatcher.main',
)]
final class LoginSuccessListener
{
    public function __invoke(LoginSuccessEvent $event): void
    {
        // ...
    }
}
```

Le dispatcher Security expose ses événements via un dispatcher par firewall (`security.event_dispatcher.<firewall_name>`). Sans `dispatcher:` explicite, le listener est attaché au dispatcher global et **n'est jamais appelé** pour ces événements.

## Pattern 3 : listener sur événement custom applicatif

Cas d'usage : un bus métier publie `OrderCancelledEvent`, un listener envoie un mail de confirmation.

```php
// src/EventListener/SendCancellationMailListener.php
declare(strict_types=1);

namespace App\EventListener;

use App\Event\OrderCancelledEvent;
use App\Mail\CancellationMailer;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

#[AsEventListener]
final class SendCancellationMailListener
{
    public function __construct(
        private readonly CancellationMailer $mailer,
    ) {}

    public function __invoke(OrderCancelledEvent $event): void
    {
        $this->mailer->send($event->getOrder());
    }
}
```

Voir `/symfony:event-dispatch` pour la définition de `OrderCancelledEvent` et son émission côté émetteur.

## Subscriber vs Listener — arbre de décision

| Situation | Choix |
|---|---|
| Un seul événement, une seule méthode | Listener `#[AsEventListener]` + `__invoke()` |
| Plusieurs événements **liés** dans une même classe (before/after filter) | Subscriber `EventSubscriberInterface` |
| Plusieurs événements **indépendants** qu'on veut regrouper | Listener avec attributs sur les méthodes |
| Classe vendor à brancher | Tag YAML `kernel.event_listener` |
| Abonnement conditionnel par environnement | YAML `when@env` ou `#[When]` |
| Bundle partagé où on veut permettre la désactivation | Listener taggué via YAML, surchargeable |
| Prioriser précisément plusieurs callbacks sur **le même** événement dans la même classe | Subscriber (forme `ExceptionEvent::class => [[...], [...]]`) |

## Vérifier l'abonnement

```bash
# tous les listeners et subscribers
symfony console debug:event-dispatcher

# un événement précis
symfony console debug:event-dispatcher "App\Event\OrderCancelledEvent"
symfony console debug:event-dispatcher kernel.exception

# dispatcher dédié (ex : security main firewall)
symfony console debug:event-dispatcher --dispatcher=security.event_dispatcher.main

# inspecter le tag posé par l'attribut
symfony console debug:container App\\EventListener\\ExceptionListener
```

Le tag `kernel.event_listener` doit apparaître avec `event`, `method`, `priority` dans la sortie de `debug:container` si l'abonnement est bien pris en compte.

## Tests

```php
declare(strict_types=1);

namespace App\Tests\EventListener;

use App\Event\OrderCancelledEvent;
use App\EventListener\SendCancellationMailListener;
use App\Mail\CancellationMailer;
use App\Order\Order;
use PHPUnit\Framework\TestCase;

final class SendCancellationMailListenerTest extends TestCase
{
    public function test_it_sends_mail_on_cancellation(): void
    {
        $order = $this->createMock(Order::class);
        $mailer = $this->createMock(CancellationMailer::class);
        $mailer->expects(self::once())->method('send')->with($order);

        (new SendCancellationMailListener($mailer))(new OrderCancelledEvent($order));
    }
}
```

Appel direct via `__invoke` — pas besoin d'instancier un `EventDispatcher`. Pour tester que l'abonnement est bien câblé, tester l'intégration (`KernelTestCase`) et dispatcher réellement l'événement pour observer le side-effect.

## Pièges fréquents

- **`#[AsEventListener]` sans type-hint ni `event:`** : Symfony ne devine plus l'événement et lève une exception au compile du container. Toujours soit typehinter le paramètre, soit passer `event:` explicitement.
- **Méthode non `public`** : l'attribut exige une méthode publique. Un `private` / `protected` est silencieusement ignoré.
- **Listener qui réassigne `$event->setResponse(...)` sans `stopPropagation()`** : les listeners suivants de priorité inférieure continuent à tourner et peuvent écraser la réponse. Si on veut imposer la réponse, appeler `$event->stopPropagation()` (cf. `/symfony:event-dispatch`).
- **Convention de nommage `on<Event>` oubliée** : sans `method:`, Symfony cherche `on<PascalCaseEventName>()`. Pour `kernel.request`, c'est `onKernelRequest` (le `.` devient majuscule). Se tromper → `LogicException: None of the "on*"/"__invoke" methods exist`.
- **Événement kernel fired mais listener silencieux** : presque toujours le tag `kernel.event_listener` manquant. `autoconfigure: true` dans `_defaults` + classe dans `src/` résolvent 95 % des cas.
- **`dispatcher:` oublié pour un événement Security / bundle tiers** : l'événement existe sur un dispatcher nommé, le listener est sur le global → jamais appelé. Vérifier dans la doc du bundle quel dispatcher est utilisé et passer le bon id.

## Déroulement

### 1 — Cadrer

- Un événement, une méthode → listener. Plusieurs liés → `/symfony:event-subscribe`.
- Événement kernel / Security / bundle / custom ? Identifier le **dispatcher** (global, security, custom).
- Priorité nécessaire ? Lancer `debug:event-dispatcher <event>` pour voir les voisins.

### 2 — Implémenter

- Classe sous `src/EventListener/<Nom>Listener.php`, `final`, `declare(strict_types=1)`.
- Préférer `__invoke()` si callback unique, sinon méthode nommée `on<Event>` ou explicite avec `method:`.
- Attribut `#[AsEventListener]` sur la classe ou sur la méthode selon le cas (`Forme 1-3`).
- `dispatcher:` si non-global.
- Constructeur avec dépendances injectées.

### 3 — Vérifier

```bash
symfony console debug:event-dispatcher <Event>::class
symfony console debug:container App\\EventListener\\NomListener
symfony console lint:container
vendor/bin/phpstan analyse src
```

Test unitaire via invocation directe, test fonctionnel si interaction avec un émetteur réel.

### 4 — Clôture

Afficher :

- Fichier créé + forme d'attribut choisie (classe / méthode, `__invoke` / nommée).
- Événement(s) et dispatcher (global ou nommé).
- Priorité et justification.
- Ce qui reste : tests, éventuellement `/symfony:event-subscribe` si l'écoute devient multi-événements, `/symfony:event-dispatch` si l'événement custom est à créer.

## Delta Sylius

- Les noms d'événements Sylius (`sylius.product.post_update`, `sylius.order.pre_create`, etc.) sont des strings. Utiliser `#[AsEventListener(event: 'sylius.product.post_update')]` et recevoir un `GenericEvent` :

```php
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\EventDispatcher\GenericEvent;

#[AsEventListener(event: 'sylius.product.post_update')]
final class ReindexProductListener
{
    public function __construct(private readonly SearchIndexer $indexer) {}

    public function __invoke(GenericEvent $event): void
    {
        /** @var \Sylius\Component\Core\Model\ProductInterface $product */
        $product = $event->getSubject();
        $this->indexer->reindex($product);
    }
}
```

- Sylius déclenche aussi des événements via le **StateMachine** (`winzou/state-machine`). Ceux-ci utilisent un mécanisme différent (callback config dans `sylius_order.yaml`, pas EventDispatcher) — si le besoin est d'intervenir sur une transition d'état, ne pas chercher un listener, mais définir un callback dans la config state machine.
- Les événements resource (`pre_*`, `post_*`) proviennent de `Sylius\Bundle\ResourceBundle\Controller\ResourceController::dispatchEvent`. Consulter le bundle pour la liste exhaustive plutôt que de deviner les noms.

## Argument optionnel

`/symfony:event-listen kernel.exception` — crée `ExceptionListener` avec `__invoke(ExceptionEvent $event)` stub, attribut `#[AsEventListener]` posé.

`/symfony:event-listen "sylius.order.post_create pour notifier Slack"` — monte un listener Sylius taggué avec `GenericEvent`.

`/symfony:event-listen src/EventListener/FooListener.php` — audit : attribut présent, méthode publique, type-hint cohérent, dispatcher correct, priorité documentée.

`/symfony:event-listen` sans argument — demande l'événement cible, le dispatcher si non-global, la logique.

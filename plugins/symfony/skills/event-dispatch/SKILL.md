---
name: event-dispatch
description: Définit et dispatche un événement Symfony custom — classe Event, EventDispatcherInterface, GenericEvent, stopPropagation. Déclenche sur "dispatcher événement", "créer event Symfony", "hook applicatif", "bus événements".
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /event-dispatch — Créer et dispatcher un événement custom

> **Utilise quand** tu crées un événement applicatif et l'émets via `EventDispatcherInterface`.
> **Pas quand** tu écoutes un événement existant → `/symfony:event-listen` (single callback) ou `/symfony:event-subscribe` (multi-callbacks).

Tu ouvres un **point d'extension** dans ton code : un moment où d'autres parties de l'application (ou de futurs bundles) peuvent brancher du comportement sans toi. Tu définis la classe d'événement, tu injectes `EventDispatcherInterface`, tu dispatches — et tu laisses les listeners / subscribers (voir `/symfony:event-listen`, `/symfony:event-subscribe`) s'y accrocher.

## Détection préalable (obligatoire)

1. Lire `composer.json` — vérifier `symfony/event-dispatcher-contracts` (tiré par `symfony/framework-bundle`). Le contrat est suffisant côté émetteur.
2. Vérifier que le cas ne se résout pas déjà avec un événement kernel existant (`kernel.response`, `kernel.controller`). Si oui, écouter l'événement framework plutôt qu'en créer un custom.
3. Se demander : **l'événement a-t-il un usage externe ?** Si personne d'autre n'écoute, un simple appel de service direct est souvent plus clair. Créer un événement n'a de sens que pour :
   - Ouvrir un point d'extension public (bundle, plugin),
   - Découpler un effet de bord transverse (audit, indexation, notification) du flux principal,
   - Offrir une sémantique before/after (permettre à des tiers de muter l'input ou d'enrichir l'output).

## Règles fondamentales

- **Une classe = un événement** : chaque événement est une classe dédiée (`OrderCreatedEvent`, `BeforeSendMailEvent`). Pas de réutilisation d'une même classe pour deux événements logiquement distincts.
- **FQCN comme identifiant** : `$dispatcher->dispatch($event)` sans deuxième argument utilise `$event::class` comme nom. Les listeners s'abonnent à `OrderCreatedEvent::class`. Ne passer un nom string que si l'événement doit aussi être routé via un alias legacy.
- **Étendre `Symfony\Contracts\EventDispatcher\Event`** et pas l'ancien `Symfony\Component\EventDispatcher\Event` (déprécié depuis 4.3). Le contrat est plus léger et découplé du composant.
- **Immutable par défaut, mutable si la sémantique l'exige** : un événement « notification » (`OrderCreatedEvent`, `AfterMailSentEvent`) est purement lecture → propriétés `readonly`. Un événement « before » (`BeforeSendMailEvent`) expose des setters pour que les listeners puissent modifier l'input.
- **Ne pas injecter de services dans l'événement** : l'événement est un **DTO** qui transporte de la donnée + éventuellement une valeur de retour. Les listeners ont leurs propres services.
- **Typage du contrat `EventDispatcherInterface`** : utiliser `Symfony\Contracts\EventDispatcher\EventDispatcherInterface` (côté émetteur), pas `Symfony\Component\EventDispatcher\EventDispatcherInterface` (legacy PSR-14). Le contrat rend le service plus portable.
- **`declare(strict_types=1)`**, classe `final` sauf si extension prévue, constructeur typehinté.

## Forme minimale : événement + dispatch

### 1 — Définir l'événement

```php
// src/Event/OrderCreatedEvent.php
declare(strict_types=1);

namespace App\Event;

use App\Entity\Order;
use Symfony\Contracts\EventDispatcher\Event;

final class OrderCreatedEvent extends Event
{
    public function __construct(
        private readonly Order $order,
    ) {}

    public function getOrder(): Order
    {
        return $this->order;
    }
}
```

Événement « notification » : lecture seule, les listeners ne modifient rien, ils réagissent.

### 2 — Dispatcher depuis un service applicatif

```php
// src/Order/OrderCreator.php
declare(strict_types=1);

namespace App\Order;

use App\Entity\Order;
use App\Event\OrderCreatedEvent;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Contracts\EventDispatcher\EventDispatcherInterface;

final class OrderCreator
{
    public function __construct(
        private readonly EntityManagerInterface $em,
        private readonly EventDispatcherInterface $dispatcher,
    ) {}

    public function create(Order $order): Order
    {
        $this->em->persist($order);
        $this->em->flush();

        $this->dispatcher->dispatch(new OrderCreatedEvent($order));

        return $order;
    }
}
```

Autowiring fait le travail : typer le constructeur sur `EventDispatcherInterface` suffit, aucun YAML à toucher.

### 3 — Côté consommateur

Un listener (`/symfony:event-listen`) ou un subscriber (`/symfony:event-subscribe`) s'abonne à `OrderCreatedEvent::class`. L'émetteur ne sait rien de qui écoute — c'est le but.

## Forme before/after : événement mutable + retour

Cas d'usage : un mailer applicatif qui permet de réécrire le sujet/message avant envoi, puis d'inspecter/modifier le retour.

### L'événement `Before` (mutable)

```php
// src/Event/BeforeSendMailEvent.php
declare(strict_types=1);

namespace App\Event;

use Symfony\Contracts\EventDispatcher\Event;

final class BeforeSendMailEvent extends Event
{
    public function __construct(
        private string $subject,
        private string $message,
    ) {}

    public function getSubject(): string { return $this->subject; }
    public function setSubject(string $subject): void { $this->subject = $subject; }

    public function getMessage(): string { return $this->message; }
    public function setMessage(string $message): void { $this->message = $message; }
}
```

### L'événement `After` (expose le retour, mutable)

```php
// src/Event/AfterSendMailEvent.php
declare(strict_types=1);

namespace App\Event;

use Symfony\Contracts\EventDispatcher\Event;

final class AfterSendMailEvent extends Event
{
    public function __construct(
        private mixed $returnValue,
    ) {}

    public function getReturnValue(): mixed { return $this->returnValue; }
    public function setReturnValue(mixed $value): void { $this->returnValue = $value; }
}
```

### Le service émetteur

```php
// src/Mail/CustomMailer.php
declare(strict_types=1);

namespace App\Mail;

use App\Event\AfterSendMailEvent;
use App\Event\BeforeSendMailEvent;
use Symfony\Contracts\EventDispatcher\EventDispatcherInterface;

final class CustomMailer
{
    public function __construct(
        private readonly EventDispatcherInterface $dispatcher,
    ) {}

    public function send(string $subject, string $message): mixed
    {
        $before = new BeforeSendMailEvent($subject, $message);
        $this->dispatcher->dispatch($before);

        // les listeners ont potentiellement muté les champs
        $result = $this->doSend($before->getSubject(), $before->getMessage());

        $after = new AfterSendMailEvent($result);
        $this->dispatcher->dispatch($after);

        return $after->getReturnValue();
    }

    private function doSend(string $subject, string $message): string
    {
        // ...
    }
}
```

Convention : `Before*Event` et `After*Event` encadrent l'action. Le `Before` porte l'input mutable, le `After` porte le retour mutable. Les listeners (`/symfony:event-listen`) peuvent par exemple censurer le message dans `Before` et enrichir le retour avec un id de tracking dans `After`.

## Propagation : `stopPropagation()` / `isPropagationStopped()`

Un listener peut stopper la chaîne — les listeners suivants (priorité inférieure) ne seront pas appelés.

```php
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

#[AsEventListener]
final class BlockSuspiciousOrderListener
{
    public function __invoke(OrderCreatedEvent $event): void
    {
        if ($this->looksFraudulent($event->getOrder())) {
            // on ne veut pas que les listeners d'audit / indexation tournent
            $event->stopPropagation();
        }
    }

    private function looksFraudulent(\App\Entity\Order $order): bool { /* ... */ }
}
```

Côté émetteur, possibilité de vérifier après le `dispatch` si un listener a stoppé la chaîne :

```php
$event = new OrderCreatedEvent($order);
$this->dispatcher->dispatch($event);

if ($event->isPropagationStopped()) {
    $this->logger->warning('OrderCreatedEvent propagation halted', ['order' => $order->getId()]);
}
```

**Règles d'usage** :

- `stopPropagation()` est un *veto* — à réserver aux cas où laisser les autres listeners tourner serait incorrect (risque de double-action, état incohérent).
- Ne pas l'utiliser pour « gagner du temps » si deux listeners sont indépendants. Le coût d'un callback est négligeable.
- Documenter côté émetteur si `isPropagationStopped()` modifie le comportement — sinon c'est un effet de bord invisible.

## `GenericEvent` : cas léger sans classe dédiée

Pour les événements trop triviaux pour mériter une classe, Symfony fournit `GenericEvent` qui transporte un `subject` + un tableau d'arguments.

```php
use Symfony\Component\EventDispatcher\GenericEvent;

$this->dispatcher->dispatch(
    new GenericEvent($order, ['reason' => 'manual_cancel']),
    'app.order.cancelled',
);
```

Côté listener :

```php
#[AsEventListener(event: 'app.order.cancelled')]
final class LogCancellationListener
{
    public function __invoke(GenericEvent $event): void
    {
        /** @var \App\Entity\Order $order */
        $order = $event->getSubject();
        $reason = $event->getArgument('reason');
    }
}
```

**Quand l'utiliser** :

- Prototype rapide avant de figer une classe dédiée.
- Interop avec Sylius qui émet presque tout en `GenericEvent`.
- Point d'extension anonyme où un string suffit comme identifiant stable.

**Quand l'éviter** :

- Contrat public / bundle : la classe dédiée documente les champs, type les accesseurs, évite les typos sur les clés d'arguments.
- Événement muté par les listeners : la classe dédiée expose des setters typés.

## Alias d'événements (`AddEventAliasesPass`)

Pour qu'un événement soit adressable à la fois par FQCN et par nom string (ex : compat avec un système legacy), déclarer un alias dans le Kernel.

```php
// src/Kernel.php
declare(strict_types=1);

namespace App;

use App\Event\MyCustomEvent;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\EventDispatcher\DependencyInjection\AddEventAliasesPass;
use Symfony\Component\HttpKernel\Kernel as BaseKernel;

class Kernel extends BaseKernel
{
    use \Symfony\Bundle\FrameworkBundle\Kernel\MicroKernelTrait;

    protected function build(ContainerBuilder $container): void
    {
        $container->addCompilerPass(new AddEventAliasesPass([
            MyCustomEvent::class => 'my_custom_event',
        ]));
    }
}
```

Conséquence : les listeners peuvent s'abonner au FQCN **ou** au string `'my_custom_event'`, les deux déclenchent le même callback. Utile pour migrer d'un nom string vers une classe typée sans casser les abonnements existants.

## Debug

```bash
# Tous les événements connus + listeners
symfony console debug:event-dispatcher

# Un événement spécifique — vérifier que le dispatch est bien reçu
symfony console debug:event-dispatcher "App\Event\OrderCreatedEvent"

# Service émetteur — vérifier que EventDispatcherInterface est injecté
symfony console debug:container App\\Order\\OrderCreator
```

`debug:event-dispatcher <event>` ne liste qu'un événement **dès qu'au moins un listener existe**. Pour vérifier qu'un dispatch muet est bien émis sans listener actif, poser temporairement un `dump($event)` dans un listener noop ou instrumenter via un décorateur de `EventDispatcherInterface`.

## Tests

### Tester l'émission côté service

```php
declare(strict_types=1);

namespace App\Tests\Order;

use App\Entity\Order;
use App\Event\OrderCreatedEvent;
use App\Order\OrderCreator;
use Doctrine\ORM\EntityManagerInterface;
use PHPUnit\Framework\TestCase;
use Symfony\Contracts\EventDispatcher\EventDispatcherInterface;

final class OrderCreatorTest extends TestCase
{
    public function test_it_dispatches_order_created_event(): void
    {
        $order = $this->createMock(Order::class);
        $em = $this->createMock(EntityManagerInterface::class);

        $dispatcher = $this->createMock(EventDispatcherInterface::class);
        $dispatcher->expects(self::once())
            ->method('dispatch')
            ->with(self::callback(
                fn (object $e) => $e instanceof OrderCreatedEvent && $e->getOrder() === $order,
            ))
            ->willReturnArgument(0);

        (new OrderCreator($em, $dispatcher))->create($order);
    }
}
```

### Tester le flux before/after avec un vrai dispatcher

```php
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();
$dispatcher->addListener(
    BeforeSendMailEvent::class,
    fn (BeforeSendMailEvent $e) => $e->setSubject('[TAGGED] '.$e->getSubject()),
);

$mailer = new CustomMailer($dispatcher);
$mailer->send('hello', 'body');
// sujet effectivement envoyé : "[TAGGED] hello"
```

Plus fidèle quand la logique dépend de la mutation par un listener.

## Pièges fréquents

- **Mauvais `EventDispatcherInterface`** : injecter `Symfony\Contracts\EventDispatcher\EventDispatcherInterface` (contrat), pas la version dans `Symfony\Component\EventDispatcher\`. Les deux existent pour compat PSR-14, mais le contrat est le bon côté émetteur.
- **`dispatch($event, $name)` avec un nom différent du FQCN** : les listeners qui s'abonnent à `OrderCreatedEvent::class` ne reçoivent rien si on dispatche avec un nom custom sans alias configuré. Préférer un seul des deux modes et rester cohérent.
- **Réutiliser un même objet événement entre deux `dispatch()`** : le flag `isPropagationStopped` reste porté par l'objet. Si on redispatch le même `$event`, les listeners ne seront pas appelés. Créer un nouvel événement à chaque émission.
- **Étendre `Symfony\Component\EventDispatcher\Event` (legacy)** : déprécié. Le linter IDE / PHPStan le signale. Basculer sur le contrat (`Symfony\Contracts\EventDispatcher\Event`).
- **`final` sur l'événement puis héritage en tests** : rare, mais si un test veut spy la classe, il faudra le retirer. Préférer `final` par défaut et spy via des mocks d'objets réels (pas d'héritage).
- **Événement utilisé comme bus de retour « forcé »** : si on se retrouve à ce que l'émetteur dépende *obligatoirement* d'un listener pour fonctionner (retour non optionnel), c'est un mauvais signal. Dans ce cas, utiliser un service avec contrat d'interface, pas un événement. L'événement est un hook optionnel, pas un câblage principal.
- **Beaucoup d'événements pour rien** : si personne n'écoute, c'est du bruit. Ouvrir un point d'extension a un coût (classe supplémentaire, test, doc). Attendre un besoin concret d'extension avant de dispatcher.

## Déroulement

### 1 — Cadrer

- Usage extérieur effectif ou hypothétique ? Si hypothétique, différer et coder l'appel direct.
- Notification (lecture seule) ou hook mutable (before / after) ?
- Classe dédiée ou `GenericEvent` suffisant ?
- Identifiant : FQCN (recommandé) ou nom string (legacy / Sylius-like) ?

### 2 — Implémenter

- `src/Event/<Nom>Event.php`, `final extends Event` (du contrat).
- Constructeur avec champs typés, `readonly` pour lecture seule, setters uniquement si mutation souhaitée.
- Injection `EventDispatcherInterface` (contrat) dans le service émetteur.
- `dispatch(new Event(...))` — pas de deuxième argument sauf si alias string.
- Si flux before/after, deux événements `BeforeX` / `AfterX`, pas un seul.

### 3 — Vérifier

```bash
symfony console debug:event-dispatcher <FQCN>
symfony console debug:container App\\Service\\NomService
symfony console lint:container
vendor/bin/phpstan analyse src
```

Test unitaire : `dispatch` appelé avec le bon événement, retour éventuel respecté. Si mutation attendue, test d'intégration avec un vrai `EventDispatcher` et un listener inline.

### 4 — Clôture

Afficher :

- Fichier(s) d'événement créé(s), service émetteur modifié.
- Sémantique : notification / before / after / mutation de retour.
- Alias posé ou non (`AddEventAliasesPass`).
- Ce qui reste : écrire les listeners (`/symfony:event-listen`) ou subscribers (`/symfony:event-subscribe`) côté consommateur ; documenter le hook dans un README ou CHANGELOG si point d'extension public.

## Delta Sylius

- Sylius dispatche **presque tout en `GenericEvent`** avec un nom string (`sylius.<resource>.<hook>`). Pour dispatcher un événement applicatif dans un projet Sylius, les deux approches coexistent :
  - **Événement applicatif propre au projet** → classe dédiée `App\Event\*` + FQCN, comme décrit plus haut. Préféré pour le code métier custom.
  - **S'intégrer aux hooks Sylius** → `GenericEvent` + nom string (`'app.<resource>.<hook>'`), cohérence avec le reste du bus Sylius, facilite l'abonnement mixte.
- Pour étendre le comportement d'un `ResourceController` Sylius (ex : après création d'un produit), **ne pas dispatcher un nouvel événement** — écouter l'événement Sylius existant (`sylius.product.post_create`, etc.) via `/symfony:event-listen` ou `/symfony:event-subscribe`.
- Les transitions de state machine Sylius (`winzou/state-machine`) ne passent **pas** par l'EventDispatcher Symfony. Pour réagir à une transition, configurer un callback dans `sylius_order.yaml` / `sylius_product_variant.yaml` — ne pas créer un événement custom pour doubler le mécanisme.

## Argument optionnel

`/symfony:event-dispatch OrderCreatedEvent` — crée la classe d'événement + montre comment l'injecter dans un service existant (à nommer ensuite).

`/symfony:event-dispatch "before/after pour CustomMailer::send()"` — monte les deux événements (`BeforeSendMailEvent`, `AfterSendMailEvent`) et patche le service pour dispatcher aux deux moments.

`/symfony:event-dispatch src/Order/OrderCreator.php` — audit : le service dispatche-t-il ? au bon endroit (après `flush` pour les notifications, avant action pour before, après pour after) ? utilise-t-il le bon contrat `EventDispatcherInterface` ?

`/symfony:event-dispatch` sans argument — demande le nom de l'événement, la sémantique (notification / before / after), les champs à transporter.

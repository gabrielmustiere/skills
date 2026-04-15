---
name: messenger-async
description: Conçoit un flux Symfony Messenger — message + handler #[AsMessageHandler], MessageBusInterface, transports (doctrine/amqp/redis), retry, DLQ, middleware. Déclenche sur "messenger", "traitement asynchrone", "queue de messages", "worker async".
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /messenger-async — Concevoir un flux Messenger (message, handler, transport, worker)

> **Utilise quand** tu déportes un traitement hors du cycle requête (envoi de mail, génération de PDF, webhook, reindex), tu structures un command/query/event bus, ou tu configures retry / DLQ / scheduler.
> **Pas quand** tu veux juste émettre un événement applicatif synchrone interne → `/symfony:event-dispatch`.
> **Pas quand** tu cherches un job CRON simple sans queue ni handler → commande Console + `symfony:scheduled-task` (ou cron système).

Tu sépares **ce qui est demandé** (le message, un DTO immutable) de **ce qui le traite** (le handler, idempotent et indépendant), reliés par un **bus** (`MessageBusInterface`) et acheminés via un **transport** (sync, doctrine, amqp, redis…). Le worker consomme, le retry rejoue les échecs transitoires, la failure transport (DLQ) capture les échecs définitifs.

## Détection préalable (obligatoire)

1. Lire `composer.json` — vérifier `symfony/messenger`. Sinon `composer require symfony/messenger`.
2. Choisir le transport selon l'infra **déjà en place** :
   - Redis dispo ? → `symfony/redis-messenger` (faible latence, simple).
   - RabbitMQ dispo ? → `symfony/amqp-messenger` (routing avancé, multi-consommateurs).
   - Aucun broker, base PostgreSQL/MySQL existante ? → `symfony/doctrine-messenger` (le plus simple à déployer, suffisant jusqu'à quelques centaines de messages/s).
   - Dev local uniquement ? → `in-memory://` (test) ou `sync://` (debug).
3. Vérifier `config/packages/messenger.yaml` — un fichier vide ou absent signifie qu'aucun bus n'est configuré, le `dispatch()` plantera silencieusement avec un message générique.
4. Vérifier `.env` pour `MESSENGER_TRANSPORT_DSN` — souvent défaut `doctrine://default?queue_name=default`.

## Règles fondamentales

- **Message = DTO immutable** : classe `final`, propriétés `readonly`, pas de service injecté, pas de logique. Sérialisable PHP par défaut (le transport sérialise/désérialise lors du fan-out).
- **Un handler par message** sauf raison explicite (effet de bord transverse type audit). Un handler = `#[AsMessageHandler]` + `__invoke(MessageType $message): void|mixed`. Le typehint résout le routage.
- **Idempotence côté handler** : un message peut être livré deux fois (worker tué, retry, ack perdu). Le handler doit produire le même résultat que la première fois — sinon prévoir une clé d'idempotence (ID de message, hash métier).
- **Pas d'objets non-sérialisables dans le message** : pas d'`EntityManager`, pas de `Request`, pas de closures. Une **entité Doctrine** est techniquement sérialisable, mais le piège est que le handler reçoit un objet **détaché** : préférer passer l'`id` et `find()` côté handler (donnée fraîche, lifecycle géré).
- **Un bus par sémantique** : `command.bus` (modifie l'état, un handler), `query.bus` (lecture, un handler, retourne via `HandledStamp`), `event.bus` (notification, N handlers). Par défaut Symfony en crée un seul (`messenger.bus.default`) — c'est suffisant tant que la séparation n'est pas explicite.
- **Routing message → transport** dans `messenger.yaml` (`routing:`). Sans entrée, le message est traité **synchrone** dans le process appelant (utile en dev / test, mais surprend en prod si oublié).
- **Worker = process long** : `bin/console messenger:consume` se gère via supervisor / systemd / `symfony:run`. Jamais lancé depuis une requête web. Toujours `--time-limit=3600 --memory-limit=128M` pour rebooter sain.
- **Failure transport obligatoire en prod** : sans, un message qui dépasse les retries est *perdu*. Configurer `failure_transport: failed` + un transport `failed` doctrine pour pouvoir inspecter / rejouer (`messenger:failed:show`, `messenger:failed:retry`).
- **Stamps = métadonnées** : `DelayStamp` (différer), `BusNameStamp` (router), `TransportNamesStamp` (forcer un transport), `HandledStamp` (récupérer le retour côté query bus). Ne pas inventer un mécanisme parallèle pour transporter ce genre d'info.

## Forme minimale : message + handler + dispatch async

### 1 — Le message (DTO)

```php
// src/Message/SendOrderConfirmation.php
declare(strict_types=1);

namespace App\Message;

final class SendOrderConfirmation
{
    public function __construct(
        public readonly int $orderId,
    ) {}
}
```

Pas l'entité `Order`, juste son `id`. Le handler rechargera depuis la DB.

### 2 — Le handler

```php
// src/MessageHandler/SendOrderConfirmationHandler.php
declare(strict_types=1);

namespace App\MessageHandler;

use App\Message\SendOrderConfirmation;
use App\Repository\OrderRepository;
use Symfony\Component\Mailer\MailerInterface;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;
use Symfony\Component\Messenger\Exception\UnrecoverableMessageHandlingException;

#[AsMessageHandler]
final class SendOrderConfirmationHandler
{
    public function __construct(
        private readonly OrderRepository $orders,
        private readonly MailerInterface $mailer,
    ) {}

    public function __invoke(SendOrderConfirmation $message): void
    {
        $order = $this->orders->find($message->orderId)
            ?? throw new UnrecoverableMessageHandlingException(
                "Order #{$message->orderId} disparue, abandon"
            );

        $this->mailer->send(/* ... */);
    }
}
```

`#[AsMessageHandler]` suffit, autoconfig wire le tag `messenger.message_handler`. Le typehint du `__invoke` résout le routage. `UnrecoverableMessageHandlingException` court-circuite le retry et envoie direct en DLQ — utile quand la cause est définitive (entité supprimée, donnée invalide).

### 3 — Le dispatch côté code applicatif

```php
// src/Order/OrderCreator.php
use App\Message\SendOrderConfirmation;
use Symfony\Component\Messenger\MessageBusInterface;

public function __construct(
    private readonly MessageBusInterface $bus,
) {}

public function create(Order $order): Order
{
    // ... persist + flush ...
    $this->bus->dispatch(new SendOrderConfirmation($order->getId()));
    return $order;
}
```

Autowiring sur `MessageBusInterface` injecte le bus par défaut. Le `dispatch()` retourne un `Envelope` — utile si on veut récupérer un stamp (ex : `HandledStamp` côté query bus).

### 4 — Le routing async

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        failure_transport: failed

        transports:
            async:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                retry_strategy:
                    max_retries: 3
                    delay: 1000
                    multiplier: 2
                    max_delay: 0
            failed: 'doctrine://default?queue_name=failed'

        routing:
            App\Message\SendOrderConfirmation: async
```

Sans la ligne `routing:`, le handler tourne dans le process appelant — la requête HTTP attend l'envoi du mail.

### 5 — Le worker

```bash
# Dev
symfony console messenger:consume async -vv

# Prod (supervisor / systemd)
bin/console messenger:consume async --time-limit=3600 --memory-limit=128M --limit=1000
```

`--time-limit` et `--memory-limit` forcent un redémarrage propre (évite les fuites cumulées). `--limit` plafonne le nombre de messages avant restart.

## Envelope + Stamps

`dispatch()` accepte un `Envelope` ou un message brut (auto-enveloppé). Pour passer des stamps :

```php
use Symfony\Component\Messenger\Envelope;
use Symfony\Component\Messenger\Stamp\DelayStamp;
use Symfony\Component\Messenger\Stamp\TransportNamesStamp;

$this->bus->dispatch(new Envelope(
    new SendOrderConfirmation($order->getId()),
    [
        new DelayStamp(60_000),               // attendre 60s avant traitement
        new TransportNamesStamp(['async']),   // forcer un transport spécifique
    ],
));
```

Stamps usuels :

- `DelayStamp(ms)` — délai avant prise en compte par le worker. Utile pour rappel, timer, retry custom.
- `TransportNamesStamp(['t1', 't2'])` — duplique vers plusieurs transports (utile fan-out manuel).
- `BusNameStamp('command.bus')` — route vers un bus précis si plusieurs configurés.
- `HandledStamp` — posé par le bus après exécution. Côté query bus, `$envelope->last(HandledStamp::class)?->getResult()` récupère le retour du handler.
- `AmqpStamp` / `RedisReceivedStamp` — métadonnées du transport, lecture seule en général.

Récupérer un stamp depuis l'envelope :

```php
$envelope = $this->bus->dispatch($message);
$delay = $envelope->last(DelayStamp::class)?->getDelay();
```

## Plusieurs bus (command / query / event)

Quand la séparation devient utile (lecture vs écriture, ou bus d'événements distinct du bus de commandes) :

```yaml
framework:
    messenger:
        default_bus: command.bus
        buses:
            command.bus:
                middleware:
                    - validation
                    - doctrine_transaction
            query.bus:
                middleware:
                    - validation
            event.bus:
                default_middleware:
                    allow_no_handlers: true   # un événement peut n'avoir aucun listener
```

Côté handler, contraindre l'attribut au bus :

```php
#[AsMessageHandler(bus: 'command.bus')]
```

Côté dispatcher, soit autowire l'interface spécifique (Symfony génère `App\\command.bus` aliasable), soit utiliser l'argument `MessageBusInterface $commandBus` (autowire par nom).

`doctrine_transaction` enveloppe le handler dans une transaction DB — tout `flush()` du handler est commit/rollback en bloc. À mettre **uniquement sur le command bus**, pas sur le query bus.

## Retry et failure transport (DLQ)

### Stratégie de retry

```yaml
transports:
    async:
        dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
        retry_strategy:
            max_retries: 3
            delay: 1000          # ms de base
            multiplier: 2        # exponentiel : 1s, 2s, 4s
            max_delay: 0         # 0 = pas de plafond
            # service: 'app.custom_retry_strategy'  # pour stratégie custom
```

Logique :

- **Erreur transitoire** (timeout API, deadlock DB) → laisser remonter l'exception, retry l'attrape, redispatch avec délai exponentiel.
- **Erreur définitive** (donnée invalide, entité supprimée) → `throw new UnrecoverableMessageHandlingException(...)` court-circuite le retry et envoie direct en DLQ.
- **Stratégie custom** : implémenter `RetryStrategyInterface` pour décisions par code HTTP, par classe d'exception, etc.

### Failure transport (Dead Letter Queue)

```yaml
framework:
    messenger:
        failure_transport: failed
        transports:
            async: '%env(MESSENGER_TRANSPORT_DSN)%'
            failed: 'doctrine://default?queue_name=failed'
```

Sans `failure_transport`, un message qui crève les retries est **perdu silencieusement** (juste loggé). Avec, il atterrit dans la table `messenger_messages` (queue `failed`) — inspectable et rejouable.

```bash
bin/console messenger:failed:show              # liste
bin/console messenger:failed:show 42 -vv       # détail (stack trace)
bin/console messenger:failed:retry             # rejoue tout en interactif
bin/console messenger:failed:retry 42          # rejoue le #42
bin/console messenger:failed:remove 42         # supprime définitivement
```

En prod : monitorer la profondeur de la queue `failed` (alerte au-delà de N messages). Une queue qui grossit = bug systématique non vu.

## Transport par cas

```yaml
transports:
    # Doctrine — simple, suffit jusqu'à ~quelques centaines msg/s
    async_doctrine: 'doctrine://default?queue_name=async&auto_setup=true'

    # Redis — faible latence, bon throughput, simple
    async_redis: '%env(MESSENGER_REDIS_DSN)%'   # redis://localhost:6379/messages

    # AMQP / RabbitMQ — routing avancé, multi-consommateurs, exchanges
    async_amqp:
        dsn: '%env(MESSENGER_AMQP_DSN)%'
        options:
            exchange:
                name: 'app'
                type: 'topic'
            queues:
                orders:
                    binding_keys: ['order.*']

    # Sync — exécution dans le process appelant (test, debug)
    sync: 'sync://'

    # In-memory — uniquement pour les tests fonctionnels
    test: 'in-memory://'
```

`auto_setup=true` (défaut) crée la table / l'exchange au premier usage. Couper en prod (`auto_setup=false`) et créer via migration / IaC pour contrôle des privilèges.

## Middleware

L'enchaînement par défaut (`add_bus_name_stamp_middleware`, `dispatch_after_current_bus`, `failed_message_processing_middleware`, `send_message`, `handle_message`) est rarement à toucher. Cas réels d'ajout :

- **`validation`** : valide le message via le composant Validator avant le handler. Utile si le message porte des contraintes (`#[Assert\NotBlank]`, etc.).
- **`doctrine_transaction`** : ouvre/commit/rollback automatiquement autour du handler. Sur le command bus uniquement.
- **Middleware custom** : implémenter `MiddlewareInterface`, `handle(Envelope $envelope, StackInterface $stack): Envelope`. Cas usuels : logging structuré, traçage OpenTelemetry, throttling applicatif, multi-tenant context propagation.

```php
final class TenantContextMiddleware implements MiddlewareInterface
{
    public function handle(Envelope $envelope, StackInterface $stack): Envelope
    {
        $stamp = $envelope->last(TenantStamp::class);
        if ($stamp !== null) {
            $this->tenantContext->switch($stamp->getTenantId());
        }
        return $stack->next()->handle($envelope, $stack);
    }
}
```

## Scheduler (Symfony 6.3+)

Pour planifier l'envoi récurrent de messages sans cron externe :

```php
use Symfony\Component\Scheduler\Attribute\AsSchedule;
use Symfony\Component\Scheduler\RecurringMessage;
use Symfony\Component\Scheduler\Schedule;
use Symfony\Component\Scheduler\ScheduleProviderInterface;

#[AsSchedule('default')]
final class MainSchedule implements ScheduleProviderInterface
{
    public function getSchedule(): Schedule
    {
        return (new Schedule())->add(
            RecurringMessage::cron('0 * * * *', new HourlyReportMessage()),
            RecurringMessage::every('5 minutes', new HealthCheckMessage()),
        );
    }
}
```

Consommer avec un transport dédié `scheduler_default` (auto-créé) et un worker :

```bash
bin/console messenger:consume scheduler_default
```

Préfère ça à un cron système pour les tâches qui sont déjà des messages — la plomberie (retry, DLQ, observabilité) suit gratuitement.

## Tests

### Test unitaire d'un handler

Le handler n'a aucune dépendance Messenger — juste les services injectés. C'est un test PHPUnit ordinaire :

```php
public function test_it_sends_confirmation(): void
{
    $orders = $this->createMock(OrderRepository::class);
    $orders->method('find')->with(42)->willReturn($order = $this->makeOrder());

    $mailer = $this->createMock(MailerInterface::class);
    $mailer->expects(self::once())->method('send');

    (new SendOrderConfirmationHandler($orders, $mailer))(
        new SendOrderConfirmation(42)
    );
}
```

### Test fonctionnel : transport `in-memory`

Dans `config/packages/test/messenger.yaml` :

```yaml
framework:
    messenger:
        transports:
            async: 'in-memory://'
```

Dans le test :

```php
use Symfony\Component\Messenger\Transport\InMemory\InMemoryTransport;

$client = static::createClient();
$client->request('POST', '/orders', [/* ... */]);

/** @var InMemoryTransport $transport */
$transport = self::getContainer()->get('messenger.transport.async');
self::assertCount(1, $transport->getSent());
self::assertInstanceOf(SendOrderConfirmation::class, $transport->getSent()[0]->getMessage());
```

### Forcer le traitement synchrone dans le test

Pour un test qui veut vérifier le résultat *après* handling sans démarrer un worker :

```yaml
framework:
    messenger:
        transports:
            async: 'sync://'
```

Le `dispatch()` exécute le handler dans le même process. Plus simple que `in-memory` quand on veut juste valider l'effet de bord en bout de chaîne.

## Debug

```bash
# Liste les messages routés et leur transport cible
symfony console debug:messenger

# Liste les transports configurés
symfony console debug:container --tag=messenger.receiver

# Inspecter la queue failed
bin/console messenger:failed:show -vv

# Stats live d'un transport doctrine
bin/console dbal:run-sql "SELECT queue_name, count(*) FROM messenger_messages GROUP BY queue_name"

# Worker en mode très verbeux pour voir chaque step
symfony console messenger:consume async -vvv
```

`debug:messenger` est le premier réflexe quand un message « part dans le vide » — souvent il est routé vers un transport qui n'a pas de worker en train de tourner.

## Pièges fréquents

- **Handler synchrone par défaut** : sans `routing:`, le message est exécuté dans le process appelant. La requête HTTP attend. Symptôme : « j'ai branché Messenger, mais ma requête est toujours lente ».
- **Entité Doctrine dans le message** : sérialisée puis reconstruite en objet **détaché** côté handler. Les relations lazy ne sont pas chargées, `flush()` plante. Toujours passer l'`id`.
- **Worker non redémarré après déploiement** : le worker garde en mémoire l'ancien code des handlers. Toujours `bin/console messenger:stop-workers` (ou `supervisorctl restart`) en post-deploy.
- **Retry sur erreur définitive** : un message avec donnée invalide retry 3× avant DLQ — bruit dans les logs, latence inutile. `UnrecoverableMessageHandlingException` court-circuite directement.
- **Queue `failed` non monitorée** : elle croît silencieusement, on découvre 50k messages échoués 6 mois plus tard. Alerter sur `count(messenger_messages WHERE queue_name='failed') > N`.
- **`auto_setup=true` en prod** : le worker crée la table avec les privilèges du compte applicatif. Préférer une migration et `auto_setup=false`.
- **Plusieurs handlers pour un même message sans le vouloir** : un copier-coller laisse deux classes avec `#[AsMessageHandler]` typant le même message. Symfony exécute les deux. `debug:messenger` les liste.
- **`dispatch()` dans une transaction Doctrine pas encore commitée** : le worker démarre, `find($id)` ne trouve pas l'entité (la transaction de l'émetteur n'est pas commit). Solutions : utiliser `DispatchAfterCurrentBusMiddleware` (par défaut sur la stack), ou émettre **après** le `flush()`.
- **Sérialisation custom oubliée** : un message avec un type complexe (enum non sérialisable PHP, objet immutable maison) plante côté worker à la désérialisation. Vérifier en envoyant un message test avant prod.
- **Worker mémoire qui explose** : sans `--memory-limit`, un leak (Doctrine UoW qui grossit, gros payload) finit par OOM. `--memory-limit=128M` + `bin/console doctrine:clear` en milieu de batch si nécessaire.

## Déroulement

### 1 — Cadrer

- **Vraiment async ?** Si le traitement < 100ms et n'est pas bloquant pour la requête, le sync direct est plus simple. Async = latence acceptable + bénéfice (parallélisme, robustesse aux pics, retry).
- **Sémantique** : commande (mute, 1 handler), query (lecture, 1 handler, retour), événement (notif, N handlers).
- **Idempotence du handler** : réfléchie avant d'écrire. Sans, le retry devient un risque.
- **Transport** : aligné sur l'infra existante. Pas de RabbitMQ « parce que c'est mieux » si on a juste Postgres.
- **Stratégie de retry** : combien de tentatives, délai, quels codes/exceptions sont définitifs.

### 2 — Implémenter

- `src/Message/<Nom>.php` — DTO `final`, propriétés `readonly`, IDs et primitives uniquement.
- `src/MessageHandler/<Nom>Handler.php` — `#[AsMessageHandler]`, `__invoke`, services injectés, `UnrecoverableMessageHandlingException` pour les cas définitifs.
- `config/packages/messenger.yaml` — transport, `routing:`, `failure_transport: failed`, retry strategy.
- `.env` / `.env.local` — `MESSENGER_TRANSPORT_DSN`.
- Côté émetteur : injecter `MessageBusInterface`, `dispatch()` après le `flush()` éventuel.
- Worker : ajouter dans supervisor / systemd / `symfony.cloud` workers.

### 3 — Vérifier

```bash
symfony console debug:messenger
symfony console messenger:consume async -vv     # voir les messages passer
symfony console messenger:failed:show           # vérifier que rien n'échoue silencieusement
vendor/bin/phpstan analyse src
```

Test : unitaire sur le handler, fonctionnel avec `in-memory` ou `sync` pour vérifier que l'émetteur dispatche bien.

### 4 — Clôture

Afficher :

- Message + handler créés, transport configuré, stratégie de retry, DLQ activée ou non.
- Routing : sync / async, transport(s) cible(s).
- Idempotence : comment elle est garantie (clé naturelle, ID, `find()` avant action).
- Ce qui reste : ajouter le worker au supervisor / au déploiement, monitorer la queue `failed`, documenter le contrat du message dans un README.

## Delta Sylius

- Sylius **n'utilise pas Messenger pour son cœur** (resources, state machine, checkout) — il s'appuie sur `winzou/state-machine` + l'EventDispatcher Symfony. Messenger reste utilisable pour le **code applicatif custom** au-dessus de Sylius (export catalogue, reindex Elasticsearch, webhooks B2B).
- Pour réagir à une transition Sylius (`sylius_order` complete, paid, cancelled), **ne pas dispatcher un message Messenger** depuis un listener — préférer un callback de state machine (`sylius_order.yaml`) qui pousse ensuite le message si traitement async nécessaire. Ça garde la single source of truth des transitions sur la state machine.
- Sylius v2 (récent) commence à intégrer Messenger pour certains flux (notifications, exports). Vérifier `composer show sylius/sylius` — si v2.x, lire `config/packages/messenger.yaml` du squelette pour ne pas dupliquer transports / routing.
- Pour les tâches CRON Sylius (clean carts abandonnés, rappels), préférer le Scheduler (Symfony 6.3+) plutôt qu'un cron système — homogène avec le reste, retry et DLQ inclus.

## Argument optionnel

`/symfony:messenger-async SendOrderConfirmation` — scaffolde le couple message + handler, ajoute le routing, n'invente pas le transport (lit la config existante).

`/symfony:messenger-async "transport doctrine + DLQ"` — patche `messenger.yaml` pour ajouter `failed` + `failure_transport`, sans toucher au reste.

`/symfony:messenger-async retry` — règle ou ajoute la `retry_strategy` sur un transport existant.

`/symfony:messenger-async scheduler` — scaffolde un `ScheduleProviderInterface` avec `#[AsSchedule]` + un message récurrent.

`/symfony:messenger-async src/MessageHandler/XHandler.php` — audit : idempotence, exceptions définitives signalées via `UnrecoverableMessageHandlingException`, pas d'entité dans le message reçu, transactions cohérentes.

`/symfony:messenger-async` sans argument — demande le cas (nouveau couple message/handler, retry, DLQ, multi-bus, scheduler, debug d'un message qui « disparaît »).

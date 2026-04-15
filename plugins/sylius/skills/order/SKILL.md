---
name: order
description: Crée ou manipule une commande Sylius par code : Order/OrderItem via factories, OrderItemQuantityModifier, CompositeOrderProcessor, applique les transitions cart→new→fulfilled. Modifier la machine elle-même → `/sylius:state-machine`.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /order — Commandes Sylius

Tu aides à **créer, manipuler ou faire transiter une commande Sylius**. Le modèle `Order` est le point de convergence de Sylius : il représente aussi bien un panier actif qu'une commande passée, tient une collection d'`OrderItem`, et porte trois state machines (état principal, shipping, payment).

Référence officielle : [docs.sylius.com/the-book/carts-and-orders/orders](https://docs.sylius.com/the-book/carts-and-orders/orders) · [checkout](https://docs.sylius.com/the-book/carts-and-orders/checkout) · [shipments](https://docs.sylius.com/the-book/carts-and-orders/shipments).

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine.
2. Vérifier `sylius/sylius` (ou `sylius/order-bundle` / `sylius/core-bundle`) dans les dépendances.
   - Présent → OK.
   - Absent → *« Ce skill cible Sylius. Je ne trouve pas `sylius/sylius` dans composer.json. Tu confirmes qu'on continue ? »*
3. Si la manipulation doit déclencher un envoi d'e-mail (confirmation, expédition) → enchaîner avec `/sylius:email`.
4. Si la demande est *« ajouter un state / une transition au cycle de vie »* (ex. nouveau state `delivered_to_relay`, retirer `skip_shipping`, brancher un listener sur `workflow.sylius_order.completed.fulfill`) → basculer sur **`/sylius:state-machine`** : ici on *applique* les transitions existantes, on ne modifie pas la machine elle-même.
5. Si la manipulation implique une remise / un ajustement manuel sur l'order → voir `/sylius:adjustment` (manuel) ou `/sylius:cart-promotion` (automatique par règles).

## Composants clés

| Composant | Service / Classe | Rôle |
|-----------|------------------|------|
| Factory Order | `sylius.factory.order` | Crée une nouvelle instance `OrderInterface` |
| Factory OrderItem | `sylius.factory.order_item` | Crée un item à rattacher à l'order |
| QuantityModifier | `sylius.modifier.order_item_quantity` | Change la quantité d'un item (recalcule les totaux) |
| OrderProcessor | `sylius.order_processing.order_processor` | Recalcule tout (prix, taxes, promos, shipping, adjustments) |
| Repository Order | `sylius.repository.order` | Persiste l'order |
| Manager Order | `sylius.manager.order` | Flush Doctrine |
| Context channel | `sylius.context.channel` | Récupère le canal courant |
| Context locale | `sylius.context.locale` | Récupère la locale courante |
| Context currency | `sylius.context.currency` | Récupère la devise courante |
| State machine factory | `sm.factory` | Construit les state machines (order, shipping, payment) |

## État du cycle de vie

| State machine | Graph | États | Transitions principales |
|---------------|-------|-------|-------------------------|
| Order principal | `sylius_order` | `cart`, `new`, `fulfilled`, `cancelled` | `create` (cart → new), `fulfill`, `cancel` |
| Shipping | `OrderShippingTransitions::GRAPH` | `cart`, `ready`, `shipped`, `cancelled` | `TRANSITION_REQUEST_SHIPPING`, `TRANSITION_SHIP`, `TRANSITION_CANCEL` |
| Payment | `OrderPaymentTransitions::GRAPH` | `cart`, `awaiting_payment`, `partially_paid`, `paid`, `cancelled`, `refunded` | `TRANSITION_REQUEST_PAYMENT`, `TRANSITION_PAY`, `TRANSITION_REFUND` |

## Règles fondamentales

- **Toujours passer par les factories** (`sylius.factory.order`, `sylius.factory.order_item`) — jamais `new Order()` direct. Les factories posent les valeurs par défaut et les events Sylius.
- **Channel + locale + currency obligatoires** avant toute opération sur un order. Sinon l'`OrderProcessor` lève silencieusement des erreurs de calcul (totaux à 0, shipping méthodes filtrées à vide).
- **Quantité via le modifier**, jamais `$orderItem->setQuantity(3)` direct — le modifier recalcule les unit prices, adjustments, stock. Même pour un item neuf à quantité 1 → passer par le modifier.
- **Appeler `OrderProcessor::process()` après chaque mutation** (ajout item, ajout shipment, changement de quantité). C'est idempotent et c'est lui qui recalcule tout l'order (totaux, taxes, promos, shipping costs, adjustments).
- **`canBeProcessed` = `state === 'cart'` par défaut**. Si tu dois retoucher un order `new` ou `fulfilled`, override cette méthode ou utilise un `OrderProcessor` dédié — ne force pas l'état.
- **State machines via `sm.factory`** : passer par le factory, jamais modifier `state` directement sur l'entité — les listeners (e-mails, stock, audit) s'abonnent aux transitions, pas aux changements de colonne.
- **Flush à la fin, pas entre chaque mutation** : accumuler, puis `manager.order->flush()` une seule fois — le `OrderProcessor` est suffisamment coûteux pour qu'on évite les flush intermédiaires.
- **Persist une fois** : `$repository->add($order)` ne doit être appelé que sur un order neuf. Les modifications ultérieures passent par le manager (déjà tracké par l'UoW Doctrine).

## Déroulement

### Cas 1 — Créer un order programmatique complet

```php
use Sylius\Component\Core\Model\OrderInterface;
use Sylius\Component\Resource\Factory\FactoryInterface;

final class CreateOrderForCustomer
{
    public function __construct(
        private FactoryInterface $orderFactory,
        private FactoryInterface $orderItemFactory,
        private OrderItemQuantityModifierInterface $quantityModifier,
        private OrderProcessorInterface $orderProcessor,
        private OrderRepositoryInterface $orderRepository,
        private ChannelContextInterface $channelContext,
        private LocaleContextInterface $localeContext,
        private CurrencyContextInterface $currencyContext,
        private ProductVariantRepositoryInterface $variantRepository,
        private CustomerRepositoryInterface $customerRepository,
    ) {}

    public function __invoke(string $customerEmail, string $variantCode, int $quantity): OrderInterface
    {
        /** @var OrderInterface $order */
        $order = $this->orderFactory->createNew();

        $order->setChannel($this->channelContext->getChannel());
        $order->setLocaleCode($this->localeContext->getLocaleCode());
        $order->setCurrencyCode($this->currencyContext->getCurrencyCode());

        $customer = $this->customerRepository->findOneBy(['email' => $customerEmail]);
        $order->setCustomer($customer);

        $variant = $this->variantRepository->findOneBy(['code' => $variantCode]);
        $orderItem = $this->orderItemFactory->createNew();
        $orderItem->setVariant($variant);
        $this->quantityModifier->modify($orderItem, $quantity);
        $order->addItem($orderItem);

        $this->orderProcessor->process($order);
        $this->orderRepository->add($order);

        return $order;
    }
}
```

Les services à injecter correspondent aux entrées de la table « Composants clés ». Les interfaces vivent dans `Sylius\Component\Core\*` et `Sylius\Component\Order\*`.

### Cas 2 — Ajouter un shipment et expédier

```php
use Sylius\Component\Core\OrderShippingTransitions;

final class ShipOrder
{
    public function __construct(
        private FactoryInterface $shipmentFactory,
        private ShippingMethodRepositoryInterface $shippingMethodRepository,
        private OrderProcessorInterface $orderProcessor,
        private StateMachineFactoryInterface $stateMachineFactory,
        private ObjectManager $orderManager,
    ) {}

    public function __invoke(OrderInterface $order, string $shippingMethodCode): void
    {
        $shipment = $this->shipmentFactory->createNew();
        $shipment->setMethod($this->shippingMethodRepository->findOneBy(['code' => $shippingMethodCode]));
        $order->addShipment($shipment);

        $this->orderProcessor->process($order);

        $stateMachine = $this->stateMachineFactory->get($order, OrderShippingTransitions::GRAPH);
        $stateMachine->apply(OrderShippingTransitions::TRANSITION_REQUEST_SHIPPING);
        $stateMachine->apply(OrderShippingTransitions::TRANSITION_SHIP);

        $this->orderManager->flush();
    }
}
```

**Adjustments de shipping** : le coût du shipment est posé comme `Adjustment` sur l'order par `OrderProcessor`, pas en dur sur le shipment — si tu inspectes les totaux, lire via `$order->getAdjustments()` filtré par type `shipping`.

### Cas 3 — Ajouter un payment et le régler

```php
use Sylius\Component\Core\OrderPaymentTransitions;

final class PayOrder
{
    public function __construct(
        private FactoryInterface $paymentFactory,
        private PaymentMethodRepositoryInterface $paymentMethodRepository,
        private StateMachineFactoryInterface $stateMachineFactory,
        private ObjectManager $orderManager,
    ) {}

    public function __invoke(OrderInterface $order, string $paymentMethodCode): void
    {
        $payment = $this->paymentFactory->createNew();
        $payment->setMethod($this->paymentMethodRepository->findOneBy(['code' => $paymentMethodCode]));
        $payment->setCurrencyCode($order->getCurrencyCode());
        $order->addPayment($payment);

        $stateMachine = $this->stateMachineFactory->get($order, OrderPaymentTransitions::GRAPH);
        $stateMachine->apply(OrderPaymentTransitions::TRANSITION_REQUEST_PAYMENT);
        $stateMachine->apply(OrderPaymentTransitions::TRANSITION_PAY);

        $this->orderManager->flush();
    }
}
```

Si plusieurs payments sont attachés à l'order, le `paymentState` de l'order passe à `partially_paid` tant que tous ne sont pas `completed`.

### Cas 4 — Annuler un order

```php
use Sylius\Component\Core\OrderTransitions;

$stateMachine = $this->stateMachineFactory->get($order, OrderTransitions::GRAPH);
if ($stateMachine->can(OrderTransitions::TRANSITION_CANCEL)) {
    $stateMachine->apply(OrderTransitions::TRANSITION_CANCEL);
    $this->orderManager->flush();
}
```

Toujours tester `can()` avant `apply()` — un order déjà `fulfilled` ne peut plus être `cancelled` par cette transition. Lever une exception métier dédiée si l'état ne permet pas l'action.

### Cas 5 — Retoucher un order existant (attention à `canBeProcessed`)

Par défaut, `OrderProcessor::process()` no-op sur un order dont `state !== 'cart'`. Pour une opération admin qui réajuste une commande `new` (ex. changer la quantité d'un item), deux options :

1. **Dédier un processor** implémentant `OrderProcessorInterface` sans la garde `canBeProcessed`, injecté explicitement.
2. **Override `Order::canBeProcessed()`** dans ton entité custom pour élargir le set d'états autorisés.

Ne **jamais** passer l'order en `cart` juste pour le re-processer puis le remettre en `new` — tu déclenches tous les listeners de `cart → new` (e-mails de confirmation renvoyés, stock réajusté, etc.).

## Pièges fréquents

- **Totaux à 0 après création** : il manque un des trois contextes (channel, locale, currency) ou tu n'as pas appelé `OrderProcessor::process()` après avoir ajouté les items. L'ordre : set contextes → add items → `process()` → `add()`.
- **Shipping méthodes vides** : `$order->getShipments()[0]->getMethod()` retourne `null` parce que la méthode n'existe pas dans ce canal. Les méthodes sont scopées par channel — vérifier `shipping_methods` de la table `sylius_channel_shipping_methods`.
- **`setQuantity()` direct au lieu du modifier** : les totaux ne sont pas recalculés et le stock n'est pas ajusté. Toujours `OrderItemQuantityModifier::modify($item, $qty)`.
- **Transition refusée silencieusement** : `apply()` lève `\SM\SMException` si la transition n'est pas possible. Toujours `can()` d'abord, ou catcher l'exception et retourner une erreur métier claire.
- **`sm.factory` manquant en Symfony 7** : Sylius est en train de migrer vers Symfony Workflow. Selon la version, il faut injecter `StateMachineInterface` (abstraction Sylius) plutôt que `sm.factory` direct. Vérifier la version de `sylius/sylius` avant d'écrire le code.
- **Guest order sans customer** : `$order->getCustomer()` peut être `null` pour un panier anonyme. Checker avant `->getEmail()` ou `->getId()`. Sylius bascule un guest en customer au moment du checkout, pas avant.
- **Double flush via OrderProcessor** : `process()` ne flush pas. Si tu appelles `flush()` entre chaque mutation, tu perds le bénéfice de l'UoW — accumuler puis `flush()` une fois à la fin.
- **Fixtures qui créent des orders en `cart`** : normal, c'est l'état par défaut. Pour simuler une commande réelle dans un fixture, faire transiter explicitement via `OrderTransitions::TRANSITION_CREATE` après avoir complété panier + shipping + payment.
- **`$order->getTotal()` après `addItem()` sans `process()`** : retourne 0. Le total est recalculé par `OrderProcessor`, pas par l'`addItem`. Ordre : add → process → read.

## Clôture

Afficher :
- Opération réalisée (création, ajout shipment, transition, etc.) et état final des trois states (`order.state`, `order.shippingState`, `order.paymentState`).
- Services injectés dans le code proposé.
- Fichiers touchés.
- Ce qui reste : fixtures de test, appels e-mail (`/sylius:email order_confirmation`), ou listeners à câbler côté métier.

## Argument optionnel

`/sylius:order create` — cadre la création d'un order programmatique complet (Cas 1).

`/sylius:order ship` — cadre l'ajout d'un shipment et les transitions shipping (Cas 2).

`/sylius:order pay` — cadre l'ajout d'un payment et les transitions payment (Cas 3).

`/sylius:order cancel` — cadre l'annulation d'un order existant (Cas 4).

`/sylius:order` sans argument — demande le cas d'usage (création, shipping, payment, cancel, retouche).

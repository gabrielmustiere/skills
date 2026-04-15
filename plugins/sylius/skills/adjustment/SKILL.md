---
name: adjustment
description: Ajoute un ajustement MANUEL Sylius (remise SAV, frais custom) sur Order, OrderItem ou OrderItemUnit via AdjustmentInterface, `lock()` pour survivre aux recalculs. Pour une remise automatique avec rules/actions → `/sylius:cart-promotion`.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /adjustment — Ajustements Sylius

Tu aides à **créer ou manipuler un Adjustment Sylius**. Un `Adjustment` est la brique qui modifie le total d'un `Order`, d'un `OrderItem` ou d'un `OrderItemUnit` : promotions, frais de port, taxes, remises manuelles. C'est l'unique surface où Sylius additionne ou retranche de la valeur à une commande.

Référence officielle : [docs.sylius.com/the-book/carts-and-orders/adjustments](https://docs.sylius.com/the-book/carts-and-orders/adjustments).

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine.
2. Vérifier `sylius/sylius` (ou `sylius/order-bundle` / `sylius/core-bundle`) dans les dépendances.
   - Présent → OK.
   - Absent → *« Ce skill cible Sylius. Je ne trouve pas `sylius/sylius` dans composer.json. Tu confirmes qu'on continue ? »*
3. Si l'adjustment est porté par une promotion → enchaîner avec `/sylius:cart-promotion` (les Promotion Adjustments sont produits par le `PromotionProcessor`, pas à la main).
4. Si l'adjustment est une remise manuelle survivant aux recalculs → prévoir systématiquement un `lock()` (sinon l'`OrderProcessor` efface l'ajustement au prochain `process()`).

## Niveaux d'application

| Niveau | Support | Méthode d'ajout | Quand l'utiliser |
|--------|---------|-----------------|------------------|
| Order | `OrderInterface` | `$order->addAdjustment($adj)` | Remise/frais qui s'applique au total global (promo panier, shipping) |
| OrderItem | `OrderItemInterface` | `$item->addAdjustment($adj)` | Remise/frais sur une ligne produit entière (toutes les unités) |
| OrderItemUnit | `OrderItemUnitInterface` | `$unit->addAdjustment($adj)` | Remise/frais sur une unité particulière (cas le plus fin, utilisé par Sylius pour les unit discounts) |

Un adjustment attaché à un niveau ne déborde **pas** sur les autres : un `OrderItem Adjustment` n'affecte pas les autres items ni le total d'order avant qu'`OrderProcessor` ne ré-agrège.

## Types d'Adjustments (`AdjustmentInterface`)

| Constante | Famille | Source usuelle |
|-----------|---------|----------------|
| `ORDER_PROMOTION_ADJUSTMENT` | Promotion | `PromotionProcessor` (action order-level) |
| `ORDER_ITEM_PROMOTION_ADJUSTMENT` | Promotion | `PromotionProcessor` (action item-level) |
| `ORDER_UNIT_PROMOTION_ADJUSTMENT` | Promotion | `PromotionProcessor` (action unit-level) |
| `ORDER_SHIPPING_PROMOTION_ADJUSTMENT` | Promotion | `PromotionProcessor` (action shipping) |
| `SHIPPING_ADJUSTMENT` | Shipping | `OrderProcessor` (coût de `Shipment`) |
| `TAX_ADJUSTMENT` | Tax | `OrderTaxationProcessor` (calculateur de taxes) |

Types custom autorisés : n'importe quelle chaîne, pas juste les constantes. Les constantes restent à privilégier pour tout ce qui a un équivalent natif — les listeners et templates admin filtrent par ces valeurs.

## Positif vs Négatif vs Neutre

| Signe | `amount` | Effet sur le total | Exemple |
|-------|----------|---------------------|---------|
| Positif | `> 0` | Augmente le total | Frais de port, taxe HT → TTC |
| Négatif | `< 0` | Diminue le total | Discount promotionnel, remise manuelle |
| Neutre | n'importe | **Aucun** effet sur `getTotal()` | Taxe déjà incluse dans le prix (affichage uniquement) |

`setNeutral(true)` = l'ajustement existe en base, est affiché dans l'UI/facture, mais n'est **pas** sommé dans le total. Cas canonique : une TVA déjà comprise dans le prix HT affiché — on veut la tracer sans double-compter.

## Composants clés

| Composant | Service / Classe | Rôle |
|-----------|------------------|------|
| Factory Adjustment | `sylius.factory.adjustment` | Crée une nouvelle `AdjustmentInterface` |
| Manager Order | `sylius.manager.order` | Flush Doctrine |
| OrderProcessor | `sylius.order_processing.order_processor` | Recalcule tout l'order ; **supprime** les adjustments non lockés avant de les reconstruire |
| PromotionProcessor | `sylius.promotion_processor` | Produit les Promotion Adjustments |
| `AdjustmentInterface` | `Sylius\Component\Order\Model\AdjustmentInterface` | Constantes de type + API (`lock`, `unlock`, `isLocked`, `isNeutral`) |

## Règles fondamentales

- **Toujours passer par `sylius.factory.adjustment`** — jamais `new Adjustment()`. La factory pose le `createdAt`, déclenche les events Sylius et câble le type d'entité attendu par Doctrine.
- **Montants en centimes (base currency)** — `setAmount(200)` = 2,00 € si la devise de base est EUR. Même convention que les prix Sylius : jamais de float, toujours des int en plus petite unité. Convertir côté applicatif (`*100`) en amont.
- **`OrderProcessor::process()` efface les adjustments non lockés**. C'est le comportement voulu : Sylius recalcule tout à chaque mutation. Un adjustment manuel qui doit survivre **doit** être locké via `$adjustment->lock()`. Sinon il disparaît au premier `process()`.
- **Les Promotion Adjustments ne se créent pas à la main**. Ils sont produits par le `PromotionProcessor` à partir d'une `Promotion` + `PromotionAction`. Créer directement un `ORDER_PROMOTION_ADJUSTMENT` contourne la traçabilité (origin, sources, compteur `used`) → passer par `/sylius:cart-promotion`.
- **Un adjustment n'est rattaché qu'à un seul niveau à la fois**. Ajouter le même objet `$adjustment` à un `Order` puis à un `OrderItem` provoque une incohérence Doctrine (`adjustable` polymorphique). Créer une instance neuve par niveau.
- **`setType()` est libre mais pas commutable** : une fois persisté, changer le type casse les filtres côté listeners et templates (`order.adjustments.type == 'tax_adjustment'`). Supprimer l'ajustement et en recréer un du bon type.
- **`setLabel()` est affiché tel quel** dans l'UI admin et certains templates front. Utiliser une chaîne stable et traduisible (clé de traduction si possible), pas du texte en dur localisé.
- **Flush après toutes les mutations**, pas entre chaque. `addAdjustment` ne persiste rien par lui-même — c'est `manager.order->flush()` qui déclenche l'INSERT.

## Déroulement

### Cas 1 — Ajouter une remise manuelle verrouillée à un order

Scénario type : geste commercial SAV, remise one-shot qui doit tenir malgré les `OrderProcessor::process()` suivants.

```php
use Sylius\Component\Order\Model\AdjustmentInterface;
use Sylius\Component\Core\Model\OrderInterface;
use Sylius\Component\Resource\Factory\FactoryInterface;
use Doctrine\Persistence\ObjectManager;

final class ApplyManualDiscount
{
    public function __construct(
        private FactoryInterface $adjustmentFactory,
        private ObjectManager $orderManager,
    ) {}

    public function __invoke(OrderInterface $order, int $amountInCents, string $label): void
    {
        /** @var AdjustmentInterface $adjustment */
        $adjustment = $this->adjustmentFactory->createNew();
        $adjustment->setType('manual_discount');
        $adjustment->setAmount(-$amountInCents); // négatif = remise
        $adjustment->setLabel($label);
        $adjustment->setNeutral(false);
        $adjustment->lock(); // survit aux recalculs

        $order->addAdjustment($adjustment);

        $this->orderManager->flush();
    }
}
```

Le `lock()` est la clé : sans lui, le prochain `OrderProcessor::process()` (déclenché par n'importe quelle mutation : ajout d'item, recalcul de shipping, application de promo) efface l'ajustement. Le `type` custom `manual_discount` évite la collision avec les constantes Sylius — laisse libre place pour filtrer en admin.

### Cas 2 — Ajustement sur un OrderItem (remise sur une ligne)

```php
use Sylius\Component\Core\Model\OrderItemInterface;

final class DiscountOneLine
{
    public function __construct(
        private FactoryInterface $adjustmentFactory,
        private ObjectManager $orderManager,
    ) {}

    public function __invoke(OrderItemInterface $item, int $amountInCents): void
    {
        $adjustment = $this->adjustmentFactory->createNew();
        $adjustment->setType('manual_item_discount');
        $adjustment->setAmount(-$amountInCents);
        $adjustment->setLabel('Remise produit défectueux');
        $adjustment->lock();

        $item->addAdjustment($adjustment);

        $this->orderManager->flush();
    }
}
```

L'ajustement impacte `$item->getTotal()` et par ricochet `$order->getTotal()` après prochain `OrderProcessor::process()`. Il n'affecte **pas** les autres items de l'order. Utile quand la remise concerne une ligne identifiée (produit abîmé, geste commercial ciblé), pas le panier entier.

### Cas 3 — Ajustement neutre (taxe déjà incluse)

```php
$adjustment = $this->adjustmentFactory->createNew();
$adjustment->setType(AdjustmentInterface::TAX_ADJUSTMENT);
$adjustment->setAmount(500); // 5,00 € de TVA incluse
$adjustment->setLabel('TVA 20 % (incluse)');
$adjustment->setNeutral(true); // n'augmente pas le total

$order->addAdjustment($adjustment);
$this->orderManager->flush();
```

L'ajustement apparaît dans `$order->getAdjustments()`, est imprimable sur facture, mais `$order->getTotal()` reste inchangé — le prix incluait déjà la taxe. C'est le mode natif du `TaxationProcessor` pour les canaux configurés en prix TTC.

### Cas 4 — Ajustement sur une OrderItemUnit

```php
use Sylius\Component\Core\Model\OrderItemUnitInterface;

/** @var OrderItemUnitInterface $unit */
$unit = $item->getUnits()->first();

$adjustment = $this->adjustmentFactory->createNew();
$adjustment->setType('manual_unit_discount');
$adjustment->setAmount(-100);
$adjustment->setLabel('Unité offerte');
$adjustment->lock();

$unit->addAdjustment($adjustment);
$this->orderManager->flush();
```

Le niveau unit sert quand une même ligne contient plusieurs exemplaires dont un seul doit être discounté (ex : « 2 achetés, le 3e à -1 € »). Sylius l'utilise nativement pour les promotions unit-level (`createUnitPercentageDiscount`).

### Cas 5 — Lister et filtrer les adjustments d'un order

```php
use Sylius\Component\Order\Model\AdjustmentInterface;

$promotionAdjustments = $order->getAdjustments(AdjustmentInterface::ORDER_PROMOTION_ADJUSTMENT);
$shippingCost        = $order->getAdjustmentsTotal(AdjustmentInterface::SHIPPING_ADJUSTMENT);
$taxes               = $order->getAdjustmentsTotalRecursively(AdjustmentInterface::TAX_ADJUSTMENT);
```

Trois accesseurs distincts :
- `getAdjustments($type = null)` — collection des ajustements du niveau courant (pas des enfants).
- `getAdjustmentsTotal($type = null)` — somme des ajustements du niveau courant.
- `getAdjustmentsTotalRecursively($type = null)` — somme remontée depuis Order → OrderItems → Units. C'est celle-ci qu'il faut pour connaître le total des taxes ou des promos de toute la commande.

### Cas 6 — Retirer un adjustment (override d'un locked)

```php
foreach ($order->getAdjustments('manual_discount') as $adjustment) {
    $adjustment->unlock(); // retire le verrou
    $order->removeAdjustment($adjustment);
}
$this->orderManager->flush();
```

`removeAdjustment()` sur un adjustment locké n'est pas bloquée, mais la sémantique correcte est d'`unlock()` d'abord — ça trace l'intention et évite qu'un process automatique se réautorise à recréer l'ajustement.

## Pièges fréquents

- **Adjustment qui disparaît au prochain `process()`** : neuf fois sur dix, `lock()` oublié. `OrderProcessor` supprime tout adjustment non locké avant de reconstruire ceux qui ont une source (promo, shipping, tax). Un ajustement manuel sans lock est considéré comme un déchet de cycle précédent.
- **Montant 10 au lieu de 1000** : les montants sont en centimes. `setAmount(10)` = 0,10 €, pas 10 €. Toujours vérifier l'échelle en croisant avec `$order->getTotal()` (lui aussi en centimes).
- **Type polymorphique incohérent** : Sylius utilise `adjustable_type` + `adjustable_id` pour le polymorphisme. Ajouter le même objet `$adjustment` à deux niveaux produit un INSERT avec un type cassé. Créer une instance neuve par niveau.
- **Créer un `ORDER_PROMOTION_ADJUSTMENT` à la main** : contourne le `PromotionProcessor`, casse le comptage `used` de la promo et le `sources` (trace d'origine) utilisé par l'admin. Pour une promo, passer par `/sylius:cart-promotion`. Pour une remise libre, utiliser un type custom.
- **`setNeutral` confondu avec `setAmount(0)`** : un ajustement neutre garde son `amount` pour l'affichage, il n'est juste pas sommé. Un ajustement à 0 est un no-op complet. Les deux s'affichent différemment en admin.
- **`getAdjustmentsTotal` au lieu de `getAdjustmentsTotalRecursively`** : sur un `Order`, le premier ne voit **pas** les adjustments posés sur les `OrderItem` ou les `Unit`. Pour un total global par type (ex. toutes les taxes de la commande), utiliser la version récursive.
- **Label en dur non traduit** : `setLabel('Remise client')` s'affiche tel quel en admin multi-locale. Utiliser une clé de traduction (`app.adjustment.manual_discount`) et traduire côté template, ou stocker un label i18n-safe.
- **Oubli du flush** : `addAdjustment` modifie l'entité en mémoire, rien n'est INSERT avant `manager.order->flush()`. Écrire un test qui relit l'order après un nouveau kernel prouve que c'est bien persisté.
- **Lock posé après `process()`** : si tu appelles `$orderProcessor->process($order)` avant `lock()`, l'adjustment a déjà été wipe. Ordre correct : créer → set → **lock** → `addAdjustment` → flush (→ éventuellement `process()` ensuite, l'adjustment survit).
- **Type custom qui casse un template** : les templates admin filtrent par type standard. Si tu introduis `type: 'manual_discount'`, prévoir un override Twig ou un event listener pour qu'il apparaisse correctement dans la facture/résumé.

## Clôture

Afficher :
- Adjustment créé (`type`, `amount` en centimes, `label`, `neutral`, `locked`).
- Niveau de rattachement (`Order` / `OrderItem` / `OrderItemUnit`) et entité cible.
- Services injectés dans le code proposé.
- Fichiers touchés.
- Ce qui reste : listener sur `OrderTransitions::TRANSITION_CANCEL` (si l'ajustement doit être révoqué à l'annulation), template admin (si type custom), tests unitaires/Behat, traduction du `label`.

## Argument optionnel

`/sylius:adjustment manual` — cadre la création d'une remise manuelle lockée sur un order (Cas 1).

`/sylius:adjustment item` — cadre un ajustement sur un `OrderItem` (Cas 2).

`/sylius:adjustment neutral` — cadre un ajustement neutre (taxe incluse, Cas 3).

`/sylius:adjustment unit` — cadre un ajustement sur une `OrderItemUnit` (Cas 4).

`/sylius:adjustment list` — cadre la lecture/filtrage des ajustements d'un order (Cas 5).

`/sylius:adjustment remove` — cadre le retrait d'un ajustement locké (Cas 6).

`/sylius:adjustment` sans argument — demande le cas d'usage (remise manuelle, item, neutre, unit, lecture, retrait).

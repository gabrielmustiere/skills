---
name: cart-promotion
description: Crée une promotion panier Sylius appliquée AUTOMATIQUEMENT : combine Rules (CartQuantity, HasTaxon, ContainsProduct) et Actions (FixedDiscount, PercentageDiscount) via PromotionProcessor. Pour un code saisi par le client → `/sylius:coupon`.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /cart-promotion — Promotions panier Sylius

Tu aides à **créer, configurer ou appliquer une promotion panier Sylius**. Une promotion Sylius combine des **règles** (conditions d'éligibilité) et des **actions** (effet du discount), orchestrées par le `PromotionProcessor` à chaque mutation du panier.

Référence officielle : [docs.sylius.com/the-book/carts-and-orders/cart-promotions](https://docs.sylius.com/the-book/carts-and-orders/cart-promotions).

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine.
2. Vérifier `sylius/sylius` (ou `sylius/promotion-bundle` / `sylius/core-bundle`) dans les dépendances.
   - Présent → OK.
   - Absent → *« Ce skill cible Sylius. Je ne trouve pas `sylius/sylius` dans composer.json. Tu confirmes qu'on continue ? »*
3. Si la promotion doit être déclenchée par un **code saisi par le client** (`PromotionCoupon`) → créer la promotion ici en `couponBased=true`, puis basculer sur **`/sylius:coupon`** pour la partie génération / application / bulk.
4. Si la demande est *« ajouter une remise manuelle, au cas par cas, hors pipeline automatique »* (SAV, ajustement one-shot sur un order existant) → basculer sur **`/sylius:adjustment`** : les Promotion Adjustments passent par le `PromotionProcessor`, les ajustements libres non.
5. Si la promotion doit mailer le client → enchaîner avec `/sylius:email`.

## Anatomie d'une promotion

| Champ | Rôle | Obligatoire |
|-------|------|-------------|
| `code` | Identifiant unique | Oui |
| `name` | Libellé affiché | Oui |
| `startsAt` / `endsAt` | Fenêtre de validité | Non (null = permanente) |
| `usageLimit` | Nombre max d'applications (tous clients confondus) | Non |
| `used` | Compteur incrémenté par le processor | Géré par Sylius |
| `exclusive` | Bloque l'application d'autres promotions | Non (défaut `false`) |
| `priority` | Ordre d'application — **nombre plus haut = appliqué en premier** | Non (défaut `0`) |
| `channels` | Canaux dans lesquels la promo est active | Oui (au moins un) |
| `rules` | Conditions à remplir | Non (sans règle = toujours éligible) |
| `actions` | Effets du discount | Oui (sinon promo no-op) |

## Composants clés

| Composant | Service / Classe | Rôle |
|-----------|------------------|------|
| Factory Promotion | `sylius.factory.promotion` | Crée une `PromotionInterface` neuve |
| Factory Rule | `sylius.factory.promotion_rule` | Helpers `createCartQuantity`, `createItemTotal`, `createHasTaxon`, etc. |
| Factory Action | `sylius.factory.promotion_action` | Helpers `createFixedDiscount`, `createPercentageDiscount`, `createUnitFixedDiscount`, `createUnitPercentageDiscount`, `createShippingPercentageDiscount` |
| Repository Promotion | `sylius.repository.promotion` | Persiste la promotion |
| PromotionProcessor | `sylius.promotion_processor` | Revert → check éligibilité → apply (appelé automatiquement par l'`OrderProcessor`) |
| PromotionApplicator | `sylius.promotion_applicator` | Applique **manuellement** une promotion à un order |
| Eligibility Checker | `sylius.promotion_eligibility_checker` | Vérifie qu'une promo est applicable (canal, dates, usage, rules) |

## Types de règles (Rule Checkers)

| Type | Méthode factory | Argument |
|------|-----------------|----------|
| Cart Quantity | `createCartQuantity($count)` | nombre d'items minimum |
| Item Total | `createItemTotal($channelCode, $amount)` | total panier en centimes |
| Has at least one from taxons | `createHasTaxon([$taxonCodes])` | liste de codes taxons |
| Total price of items from taxon | `createTaxonTotal([...])` | taxon + seuil par canal |
| Nth Order | `createNthOrder($n)` | rang de la commande |
| Shipping Country | `createShippingCountry($countryCode)` | code ISO pays |
| Customer Group | `createCustomerGroup($groupCode)` | code du customer group |
| Contains Product | `createContainsProduct($productCode)` | code produit |

## Types d'actions (Discounts)

| Type | Méthode factory | Effet |
|------|-----------------|-------|
| Order Fixed | `createFixedDiscount($amount, $channelCode)` | Montant fixe sur le total order (en centimes) |
| Order Percentage | `createPercentageDiscount($ratio)` | Pourcentage sur le total order (`0.10` = 10 %) |
| Unit Fixed (Item) | `createUnitFixedDiscount($amount, $channelCode)` | Montant fixe par unité |
| Unit Percentage (Item) | `createUnitPercentageDiscount($ratio)` | Pourcentage par unité |
| Shipping Percentage | — configurer `type: shipping_percentage_discount` sur l'action | Pourcentage sur les frais de port |

## Règles fondamentales

- **Toujours passer par les factories** (`sylius.factory.promotion`, `sylius.factory.promotion_rule`, `sylius.factory.promotion_action`). Les factories posent les `type`, `configuration` et évènements Sylius — un `new Promotion()` direct produit une entité incomplète.
- **Rattacher au moins un channel** via `addChannel($channel)` — sinon `PromotionEligibilityChecker` rejette la promotion systématiquement.
- **Rules sans Actions = promo no-op.** Une promotion sans action est techniquement éligible mais ne modifie rien. Toujours ajouter au moins une action avant `repository->add()`.
- **Montants en centimes.** Les API `createFixedDiscount(1000, 'US_WEB')` signifient 10 $, pas 1000 $. Convertir côté applicatif (`*100`) en amont.
- **`priority` plus haut = appliqué plus tôt.** Pour qu'un pourcentage s'applique avant un montant fixe, lui donner une `priority` supérieure. Les promotions **exclusives** coupent court : la première exclusive applicable ferme la chaîne.
- **Ne jamais appeler `PromotionProcessor` à la main** sur un order dont l'`OrderProcessor` est déjà en cours (boucle infinie). L'`OrderProcessor` délègue au `PromotionProcessor` dans son pipeline — se contenter d'appeler `orderProcessor->process($order)`.
- **`PromotionApplicator::apply()` = contournement.** Sert à forcer une promotion hors du cycle normal (back-office, commande legacy). Ne pas l'utiliser pour l'eligibility-check standard — c'est `PromotionProcessor` qui gère.
- **Filtres = scope de l'action, pas de la règle.** Un `TaxonFilter` sur une action `createUnitPercentageDiscount` applique 10 % uniquement aux items de ce taxon, mais la règle d'éligibilité reste globale.
- **`usageLimit` vs `perCustomerUsageLimit`** : le premier est global, le second nécessite que l'order ait un `customer` rattaché — sinon la limite par client n'est pas comptabilisée.

## Déroulement

### Cas 1 — Créer une promotion « 10 % sur tout le panier »

```php
use Sylius\Component\Core\Model\PromotionInterface;
use Sylius\Component\Promotion\Factory\PromotionRuleFactoryInterface;
use Sylius\Component\Promotion\Factory\PromotionActionFactoryInterface;
use Sylius\Component\Resource\Factory\FactoryInterface;
use Sylius\Component\Resource\Repository\RepositoryInterface;

final class CreateTenPercentPromotion
{
    public function __construct(
        private FactoryInterface $promotionFactory,
        private PromotionActionFactoryInterface $actionFactory,
        private RepositoryInterface $promotionRepository,
        private ChannelRepositoryInterface $channelRepository,
    ) {}

    public function __invoke(string $channelCode): PromotionInterface
    {
        /** @var PromotionInterface $promotion */
        $promotion = $this->promotionFactory->createNew();
        $promotion->setCode('ten_percent_off');
        $promotion->setName('10 % sur le panier');
        $promotion->setPriority(10);
        $promotion->addChannel($this->channelRepository->findOneByCode($channelCode));

        $action = $this->actionFactory->createPercentageDiscount(0.10);
        $promotion->addAction($action);

        $this->promotionRepository->add($promotion);

        return $promotion;
    }
}
```

### Cas 2 — Promotion avec règle « minimum 5 items »

```php
final class CreateBulkDiscount
{
    public function __construct(
        private FactoryInterface $promotionFactory,
        private PromotionRuleFactoryInterface $ruleFactory,
        private PromotionActionFactoryInterface $actionFactory,
        private RepositoryInterface $promotionRepository,
        private ChannelRepositoryInterface $channelRepository,
    ) {}

    public function __invoke(string $channelCode): void
    {
        /** @var PromotionInterface $promotion */
        $promotion = $this->promotionFactory->createNew();
        $promotion->setCode('bulk_5_items');
        $promotion->setName('5 articles → 10 € offerts');
        $promotion->addChannel($this->channelRepository->findOneByCode($channelCode));

        $promotion->addRule($this->ruleFactory->createCartQuantity(5));
        $promotion->addAction($this->actionFactory->createFixedDiscount(1000, $channelCode));

        $this->promotionRepository->add($promotion);
    }
}
```

Le montant `1000` est en centimes (10,00 €). `createFixedDiscount` exige le `channelCode` car un montant fixe est toujours lié à une devise, donc à un canal.

### Cas 3 — Promotion ciblée sur un taxon

Règle d'éligibilité + filtre sur l'action, combinés :

```php
$promotion->addRule($this->ruleFactory->createHasTaxon(['summer_sale']));

$action = $this->actionFactory->createUnitPercentageDiscount(0.20);
$action->setConfiguration(array_merge($action->getConfiguration(), [
    'filters' => [
        'taxons_filter' => ['taxons' => ['summer_sale']],
    ],
]));
$promotion->addAction($action);
```

La règle `HasTaxon` rend la promo éligible dès qu'**au moins un** item du panier est dans `summer_sale`. Le filtre `taxons_filter` sur l'action garantit que le discount s'applique **uniquement** à ces items, pas au reste du panier.

### Cas 4 — Promotion exclusive avec priorité

```php
$promotion->setExclusive(true);
$promotion->setPriority(100);
```

Pose deux invariants :
- Si `exclusive=true`, aucune autre promotion ne peut s'appliquer sur le même order (même si elles sont éligibles).
- `priority=100` la fait passer avant toute promo de priorité inférieure — critique quand plusieurs exclusives sont éligibles : la plus prioritaire gagne et court-circuite les autres.

### Cas 5 — Appliquer une promotion manuellement (hors cycle)

```php
/** @var PromotionApplicatorInterface $applicator */
$applicator = $this->container->get('sylius.promotion_applicator');
$applicator->apply($order, $promotion);
```

Réservé aux cas où le `PromotionProcessor` n'est pas déclenché (script de migration, correctif manuel). Pour le flux standard, c'est `OrderProcessor::process()` qui pilote l'application.

### Cas 6 — Fenêtre de validité et usage limit

```php
$promotion->setStartsAt(new \DateTimeImmutable('2026-05-01'));
$promotion->setEndsAt(new \DateTimeImmutable('2026-05-31 23:59:59'));
$promotion->setUsageLimit(100);
```

Après 100 applications (tous canaux confondus), `PromotionEligibilityChecker` écarte la promo. Le compteur `used` est incrémenté par le processor à chaque commande qui la consomme — attention, un order annulé ne décrémente pas automatiquement le compteur (il faut un listener custom sur `OrderTransitions::TRANSITION_CANCEL`).

## Pièges fréquents

- **Promotion ignorée malgré des rules satisfaites** : neuf fois sur dix, le `channel` n'est pas rattaché. Sans `addChannel()`, la promo n'est jamais éligible. Vérifier `SELECT * FROM sylius_promotion_channels WHERE promotion_id = ...`.
- **`createFixedDiscount` sans channel code** : la méthode exige `(amount, channelCode)`. Oublier le deuxième argument produit une config invalide silencieusement — la promo s'enregistre mais ne s'applique sur aucun canal.
- **Discount appliqué sur le mauvais montant** : les pourcentages prennent un **float entre 0 et 1** (`0.10` = 10 %). Passer `10` applique 1 000 %. Vérifier systématiquement l'échelle.
- **Promotions qui se cumulent de façon imprévue** : sans `setExclusive(true)`, toutes les promos éligibles s'empilent. Si deux `createPercentageDiscount(0.10)` sont actives, on obtient -20 % en cumul — passer en exclusif ou jouer sur `priority` selon l'effet voulu.
- **Filter appliqué à la règle au lieu de l'action** : la règle `HasTaxon` ne filtre **pas** les items éligibles au discount, elle ne fait qu'ouvrir la promo. Le filtre doit être posé sur l'action (`filters` dans `configuration`).
- **`usageLimit` bloquant en test** : les fixtures qui rejouent la même promo atteignent la limite. En env de dev, laisser `usageLimit=null` ou reset la colonne `used` avant chaque run.
- **Promotion exclusive masquée par une autre plus prioritaire** : si deux exclusives sont éligibles, seule celle de plus haute `priority` s'applique. Documenter explicitement les priorités quand plusieurs exclusives coexistent.
- **Coupons confondus avec promotions** : une promotion avec `couponBased=true` ne s'applique que si un `PromotionCoupon` valide est posé sur l'order (`$order->setPromotionCoupon($coupon)`). Sinon elle reste inerte, même éligible côté rules.
- **Compteur `used` qui ne redescend jamais** : par design, Sylius ne revert pas le compteur sur un order annulé. Pour les promotions à quota strict, ajouter un listener sur `OrderTransitions::TRANSITION_CANCEL` qui décrémente `setUsed($promotion->getUsed() - 1)`.
- **Filtre taxon qui matche tous les items enfants** : `TaxonFilter` descend récursivement dans l'arbre des taxons. Pour cibler un taxon feuille sans ses descendants, passer par un taxon dédié ou un filtre custom.

## Clôture

Afficher :
- Promotion créée (`code`, `name`, canaux, `priority`, `exclusive`, rules, actions).
- Services injectés dans le code proposé.
- Fichiers touchés.
- Ce qui reste : fixtures (`PromotionFixture`), tests Behat (`features/promotion/*`), listeners à câbler (ex. décrément du `used` à l'annulation), ou e-mail de notification (`/sylius:email`).

## Argument optionnel

`/sylius:cart-promotion create` — cadre la création d'une promotion simple sans règles (Cas 1).

`/sylius:cart-promotion rule` — cadre l'ajout de règles d'éligibilité (Cas 2).

`/sylius:cart-promotion taxon` — cadre une promotion ciblée sur un taxon avec filtre (Cas 3).

`/sylius:cart-promotion exclusive` — cadre la configuration d'exclusivité et priorité (Cas 4).

`/sylius:cart-promotion apply` — cadre l'application manuelle hors cycle (Cas 5).

`/sylius:cart-promotion` sans argument — demande le cas d'usage (création, règle, filtre taxon, exclusivité, application manuelle, fenêtre de validité).

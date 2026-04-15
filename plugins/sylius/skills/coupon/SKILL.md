---
name: coupon
description: Crée, applique ou génère en masse des codes promo Sylius (PromotionCoupon) saisis par le client — expirationDate, usageLimit, bulk via PromotionCouponGenerator. La Promotion (`couponBased=true`) d'abord → `/sylius:cart-promotion`.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /coupon — Coupons de promotion Sylius

Tu aides à **créer, appliquer ou générer en masse des coupons Sylius**. Un `PromotionCoupon` est un code unique rattaché à une `Promotion` marquée `couponBased=true` : la promotion ne s'applique que si l'order porte un coupon valide (`$order->setPromotionCoupon($coupon)`).

Référence officielle : [docs.sylius.com/the-book/carts-and-orders/coupons](https://docs.sylius.com/the-book/carts-and-orders/coupons).

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine.
2. Vérifier `sylius/sylius` (ou `sylius/promotion-bundle` / `sylius/core-bundle`) dans les dépendances.
   - Présent → OK.
   - Absent → *« Ce skill cible Sylius. Je ne trouve pas `sylius/sylius` dans composer.json. Tu confirmes qu'on continue ? »*
3. Si la promotion support n'existe pas encore → enchaîner avec `/sylius:cart-promotion` pour la créer en mode `couponBased=true` avant de câbler les coupons.
4. Si l'envoi du code par e-mail est nécessaire → enchaîner avec `/sylius:email`.

## Anatomie d'un coupon

| Champ | Rôle | Obligatoire |
|-------|------|-------------|
| `code` | Identifiant unique du coupon (valeur tapée par le client) | Oui |
| `expiresAt` / `expirationDate` | Date d'expiration (null = permanent) | Non |
| `usageLimit` | Nombre max total d'utilisations (tous clients confondus) | Non |
| `perCustomerUsageLimit` | Quota par `customer` (nécessite un client rattaché à l'order) | Non |
| `used` | Compteur incrémenté par le `PromotionProcessor` à chaque commande qui consomme le coupon | Géré par Sylius |
| `promotion` | Promotion parente (doit être `couponBased=true`) | Oui |

## Composants clés

| Composant | Service / Classe | Rôle |
|-----------|------------------|------|
| Factory Coupon | `sylius.factory.promotion_coupon` | Crée un `PromotionCouponInterface` neuf |
| Repository Coupon | `sylius.repository.promotion_coupon` | Persiste / recherche les coupons (souvent par `code`) |
| Factory Promotion | `sylius.factory.promotion` | Crée la promotion parente (marquée `couponBased=true`) |
| Repository Promotion | `sylius.repository.promotion` | Persiste la promotion (avec la collection de coupons cascade) |
| Coupon Generator | `sylius.promotion_coupon_generator` | Génère N coupons uniques pour une promotion donnée |
| Generator Instruction | `PromotionCouponGeneratorInstruction` | DTO : `amount`, `codeLength`, `prefix`, `suffix`, `expiresAt`, `usageLimit` |
| OrderProcessor | `sylius.order_processing.order_processor` | Recalcule l'order et délègue au `PromotionProcessor` qui revalide le coupon |
| Eligibility Checker | `sylius.promotion_coupon_eligibility_checker` | Vérifie validité (code, expiration, usage, promotion active) |

## Règles fondamentales

- **Toujours passer par les factories** (`sylius.factory.promotion_coupon`, `sylius.factory.promotion`). Un `new PromotionCoupon()` direct produit une entité détachée, non traçable par le processor.
- **La promotion parente doit être `couponBased=true`.** Sans ce flag, `addCoupon()` lève une `InvalidArgumentException` côté Sylius et la promo reste inerte même si un code est saisi.
- **Un coupon = une promotion.** La relation est many-to-one côté coupon (plusieurs coupons peuvent référencer une même promo, mais un coupon n'appartient qu'à une promo).
- **Le code du coupon est l'identifiant public.** Il doit être **unique globalement** (contrainte DB sur `sylius_promotion_coupon.code`). Les collisions font échouer le flush Doctrine.
- **`expiresAt` est vérifié à l'application, pas à la création.** Un coupon expiré reste en base mais est rejeté par le `PromotionCouponEligibilityChecker` — utile pour l'historique.
- **`usageLimit` vs `perCustomerUsageLimit`** : global vs par client. Le second nécessite que `$order->getCustomer()` soit non null ; sinon Sylius ne peut pas compter et la limite par client est ignorée silencieusement.
- **Application via l'order, pas le coupon.** `$order->setPromotionCoupon($coupon)` + `OrderProcessor::process()` est le seul chemin standard. Ne jamais appeler `PromotionApplicator` pour un coupon — il court-circuite la vérification d'éligibilité du coupon.
- **Le compteur `used` ne redescend pas** sur un order annulé. Prévoir un listener sur `OrderTransitions::TRANSITION_CANCEL` si la limite est critique.
- **Le Generator produit des codes aléatoires alphanumériques majuscules.** `codeLength` par défaut = 6. Avec `prefix="NY_"` et `codeLength=8`, on obtient `NY_A3F9K2LM`.

## Déroulement

### Cas 1 — Créer un coupon unique sur une promotion existante

```php
use Sylius\Component\Core\Model\PromotionInterface;
use Sylius\Component\Core\Model\PromotionCouponInterface;
use Sylius\Component\Resource\Factory\FactoryInterface;
use Sylius\Component\Resource\Repository\RepositoryInterface;

final class CreateFreeShippingCoupon
{
    public function __construct(
        private FactoryInterface $couponFactory,
        private RepositoryInterface $promotionRepository,
    ) {}

    public function __invoke(): PromotionCouponInterface
    {
        /** @var PromotionInterface $promotion */
        $promotion = $this->promotionRepository->findOneBy(['code' => 'free_shipping']);

        if (!$promotion->isCouponBased()) {
            throw new \DomainException('La promotion doit être couponBased=true.');
        }

        /** @var PromotionCouponInterface $coupon */
        $coupon = $this->couponFactory->createNew();
        $coupon->setCode('FREESHIPPING2026');
        $coupon->setExpiresAt(new \DateTime('2026-12-31 23:59:59'));
        $coupon->setUsageLimit(500);

        $promotion->addCoupon($coupon);
        // Flush via le manager Doctrine (cascade persist configurée côté Sylius).
        $this->promotionRepository->add($promotion);

        return $coupon;
    }
}
```

Le `add()` sur le repository de la promotion déclenche la cascade sur la collection `coupons` — pas besoin de flusher le coupon séparément.

### Cas 2 — Créer promotion coupon-based + coupon en un seul flux

```php
use Sylius\Component\Promotion\Factory\PromotionActionFactoryInterface;

final class CreateBlackFridayCoupon
{
    public function __construct(
        private FactoryInterface $promotionFactory,
        private FactoryInterface $couponFactory,
        private PromotionActionFactoryInterface $actionFactory,
        private RepositoryInterface $promotionRepository,
        private ChannelRepositoryInterface $channelRepository,
    ) {}

    public function __invoke(string $channelCode): void
    {
        /** @var PromotionInterface $promotion */
        $promotion = $this->promotionFactory->createNew();
        $promotion->setCode('black_friday_2026');
        $promotion->setName('Black Friday 2026');
        $promotion->setCouponBased(true);
        $promotion->addChannel($this->channelRepository->findOneByCode($channelCode));
        $promotion->addAction($this->actionFactory->createPercentageDiscount(0.25));

        /** @var PromotionCouponInterface $coupon */
        $coupon = $this->couponFactory->createNew();
        $coupon->setCode('BF25');
        $coupon->setUsageLimit(1000);
        $coupon->setPerCustomerUsageLimit(1);

        $promotion->addCoupon($coupon);

        $this->promotionRepository->add($promotion);
    }
}
```

`perCustomerUsageLimit=1` garantit qu'un même client ne peut utiliser `BF25` qu'une seule fois — à condition que `$order->getCustomer()` soit non null au moment de l'application.

### Cas 3 — Appliquer un coupon sur un order

```php
use Sylius\Component\Core\OrderProcessing\OrderProcessorInterface;

final class ApplyCouponToOrder
{
    public function __construct(
        private RepositoryInterface $couponRepository,
        private OrderProcessorInterface $orderProcessor,
    ) {}

    public function __invoke(OrderInterface $order, string $couponCode): void
    {
        /** @var PromotionCouponInterface|null $coupon */
        $coupon = $this->couponRepository->findOneBy(['code' => $couponCode]);

        if (null === $coupon) {
            throw new \DomainException(sprintf('Coupon %s introuvable.', $couponCode));
        }

        $order->setPromotionCoupon($coupon);
        $this->orderProcessor->process($order);
    }
}
```

L'`OrderProcessor` relance tout le pipeline : il appelle le `PromotionProcessor` qui revalide l'éligibilité du coupon (expiration, usage, promotion active, channel) avant d'appliquer l'action. Si le coupon est invalide, il est silencieusement ignoré — l'order reste sans discount.

### Cas 4 — Générer 50 coupons en masse

```php
use Sylius\Component\Promotion\Generator\PromotionCouponGeneratorInterface;
use Sylius\Component\Promotion\Model\PromotionCouponGeneratorInstruction;

final class BulkGenerateNewYearCoupons
{
    public function __construct(
        private RepositoryInterface $promotionRepository,
        private PromotionCouponGeneratorInterface $couponGenerator,
    ) {}

    public function __invoke(): void
    {
        /** @var PromotionInterface $promotion */
        $promotion = $this->promotionRepository->findOneBy(['code' => 'new_year_sale']);

        $instruction = new PromotionCouponGeneratorInstruction();
        $instruction->setAmount(50);
        $instruction->setCodeLength(8);
        $instruction->setPrefix('NY26_');
        $instruction->setSuffix('_VIP');
        $instruction->setExpiresAt(new \DateTime('2026-01-31'));
        $instruction->setUsageLimit(1);

        $this->couponGenerator->generate($promotion, $instruction);
    }
}
```

Le générateur garantit l'unicité **dans la promotion cible**, pas globalement — en théorie deux promotions distinctes peuvent produire le même suffixe aléatoire, mais la contrainte d'unicité DB sur `code` fera échouer le second flush. Anticiper en ajoutant un `prefix` distinct par promotion.

### Cas 5 — Lister les coupons d'une promotion

```php
$promotion = $this->promotionRepository->findOneBy(['code' => 'black_friday_2026']);

foreach ($promotion->getCoupons() as $coupon) {
    printf("%s — utilisé %d/%s fois\n",
        $coupon->getCode(),
        $coupon->getUsed(),
        $coupon->getUsageLimit() ?? '∞'
    );
}
```

`getCoupons()` renvoie une `Collection` Doctrine. Pour des volumes > quelques centaines, préférer une requête DQL paginée sur `sylius.repository.promotion_coupon` avec filtre `promotion = :promo`.

## Pièges fréquents

- **Coupon sans promotion `couponBased=true`** : le coupon se crée mais n'est jamais pris en compte — `PromotionCouponEligibilityChecker` ignore le coupon si la promo parente n'est pas marquée. Vérifier `SELECT coupon_based FROM sylius_promotion WHERE id = ...`.
- **Code dupliqué** : deux coupons avec le même `code` (même sur deux promotions différentes) → violation de contrainte unique sur `sylius_promotion_coupon.code`. Toujours préfixer par promotion ou passer par le Generator.
- **`perCustomerUsageLimit` silencieusement ignoré** : sans `$order->setCustomer()`, Sylius ne peut pas indexer l'usage par client. Vérifier que le checkout rattache bien le customer avant d'appliquer le coupon.
- **Coupon expiré mais encore appliqué** : le `PromotionCouponEligibilityChecker` rejette, mais si tu appelles `PromotionApplicator::apply()` directement, la vérification coupon est court-circuitée. Toujours passer par `OrderProcessor::process()`.
- **Compteur `used` qui explose en test** : les fixtures rejouent la même commande → le coupon atteint `usageLimit`. En dev, laisser `usageLimit=null` ou reset `sylius_promotion_coupon.used` entre les runs.
- **Prefix/suffix qui dépasse la longueur en BDD** : la colonne `code` est un `VARCHAR(255)` par défaut, mais les fixtures et les UI coupent souvent à 32. Rester sous 32 caractères totaux (prefix + codeLength + suffix).
- **Generator appelé sur une promotion non coupon-based** : `PromotionCouponGenerator::generate()` lève une exception. Vérifier `$promotion->isCouponBased()` avant.
- **Order annulé → coupon toujours marqué utilisé** : comme pour les promotions, `used` ne se décrémente pas à l'annulation. Listener sur `OrderTransitions::TRANSITION_CANCEL` indispensable pour les coupons à quota strict (campagnes influenceur, cadeaux nominatifs).
- **Plusieurs coupons sur un même order** : `Order` ne porte qu'**un seul** `promotionCoupon`. Pour cumuler plusieurs codes, il faut soit des promotions non coupon-based, soit un module tiers (ex. `setono/sylius-coupons-plugin`).
- **Coupons supprimés mais référencés par des orders** : la FK `order.promotion_coupon_id` peut pointer vers un coupon supprimé si la cascade n'est pas configurée. Préférer `expiresAt` dans le passé à une suppression hard.

## Clôture

Afficher :
- Coupon(s) créé(s) (`code`, `expiresAt`, `usageLimit`, `perCustomerUsageLimit`, promotion rattachée).
- Services injectés dans le code proposé.
- Fichiers touchés.
- Ce qui reste : fixtures (`PromotionCouponFixture`), tests Behat (`features/promotion/using_coupon_*`), listeners à câbler (décrément de `used` à l'annulation), envoi d'e-mail du code (`/sylius:email`), endpoint d'application du coupon côté checkout (`ApplyCouponAction`).

## Argument optionnel

`/sylius:coupon create` — cadre la création d'un coupon unique sur une promotion existante (Cas 1).

`/sylius:coupon promotion` — cadre la création d'une promotion coupon-based + coupon dans le même flux (Cas 2).

`/sylius:coupon apply` — cadre l'application d'un coupon sur un order via `OrderProcessor` (Cas 3).

`/sylius:coupon generate` — cadre la génération bulk via `PromotionCouponGenerator` (Cas 4).

`/sylius:coupon list` — cadre la lecture des coupons d'une promotion (Cas 5).

`/sylius:coupon` sans argument — demande le cas d'usage (création unitaire, promotion + coupon, application, génération bulk, listing).

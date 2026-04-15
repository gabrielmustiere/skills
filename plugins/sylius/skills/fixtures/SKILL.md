---
name: fixtures
description: Customise les fixtures Sylius : modifie la suite `default` dans `sylius_fixtures.yaml` (currencies, channels, shipping/payment methods), étend un `ExampleFactory` + `Fixture` pour un champ custom. Champ absent → `/sylius:model` d'abord.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /fixtures — Customiser les fixtures Sylius

Tu aides à **customiser les fixtures Sylius** : soit pour **modifier la suite `default`** (currencies, channels, shipping/payment methods, etc.) via `config/packages/sylius_fixtures.yaml`, soit pour **étendre un `ExampleFactory` + `Fixture`** afin d'exposer un champ ajouté sur une entité customisée. Les fixtures sont des objets PHP qui implémentent `Sylius\Bundle\FixturesBundle\Fixture\FixtureInterface` et sont taguées `sylius_fixtures.fixture` ; elles servent à initialiser ou modifier l'état de l'application (DB, fichiers, événements) — usage typique : seed dev/QA, demo data, init prod.

Référence officielle : [docs.sylius.com/the-customization-guide/customizing-fixtures](https://docs.sylius.com/the-customization-guide/customizing-fixtures).

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine.
2. Vérifier `sylius/sylius` dans les dépendances.
   - Présent → OK.
   - Absent → *« Ce skill cible Sylius (customisation des fixtures via `sylius_fixtures.suites.*`). Je ne trouve pas `sylius/sylius`. On continue quand même ? »*
3. Si la demande consiste à **ajouter un champ à un modèle natif** (ex. `deliveryConditions` sur `ShippingMethod`) avant de l'exposer en fixture → basculer d'abord sur **`/sylius:model`** (ou **`/sylius:translation-entity`** si traduisible) pour créer le champ + migration, **puis revenir ici** pour étendre la factory et la fixture.
4. Si la demande est *« charger des fixtures de test pour un test fonctionnel/E2E »* (et pas customiser les fixtures elles-mêmes) → c'est l'utilisation de fixtures, pas leur customisation : `sylius:fixtures:load <suite>` suffit, pas besoin de cette skill.

## Règles fondamentales

- **Suite `default` = la suite chargée par `sylius:fixtures:load` sans argument**. Les autres suites (`dev`, `test`, `demo`…) doivent être appelées explicitement : `sylius:fixtures:load demo`. Utile pour séparer seed prod (`default`) et seed QA/demo.
- **`priority`** : **plus le nombre est petit, plus la fixture s'exécute tard**. C'est l'inverse de l'intuition classique. Ordre par défaut implicite via dépendances déclarées dans les fixtures vendor (currencies avant channels, channels avant shipping methods, etc.) — ne pas y toucher sans raison forte.
- **Clé `custom`** : ajoute des entrées **sans remplacer** les fixtures existantes par défaut de Sylius. Si tu veux **ne charger que tes entrées**, il faut désactiver la suite par défaut et ne déclarer que les tiennes — sinon les channels/products fashion store coexisteront avec les tiens.
- **Le tag `sylius_fixtures.fixture` est obligatoire** : sans lui, la classe est instanciée mais jamais découverte par le `FixtureRegistry`. Vérifier via `bin/console debug:container --tag=sylius_fixtures.fixture` que ta fixture apparaît bien.
- **Constructeur de l'`ExampleFactory`** : depuis Sylius 2.0.8, les arguments sont `protected` → on peut juste appeler `parent::__construct(...)` et n'override que ce qui change. Sur Sylius < 2.0.8, **il faut redéclarer le constructeur complet** avec tous les arguments du parent (sinon l'autowiring échoue ou des dépendances manquent).
- **Override du service Sylius natif** : on **redéfinit le service avec le même id** (`sylius.fixture.example_factory.shipping_method`) — pas un alias, pas un nouveau service. Le tag `public: true` est requis pour que le `FixtureBuilder` puisse le résoudre via le container.
- **Désactiver l'autowiring sur `src/Fixture/`** : sans exclusion, Symfony auto-configure tes fixtures **en plus** des services manuellement déclarés → `Multiple definitions for the same service id`. La règle d'exclusion vit dans `config/services.yaml` :
  ```yaml
  App\:
      resource: '../src/*'
      exclude: '../src/{Entity,Fixture,Migrations,Tests,Kernel.php}'
  ```
- **Translatable + fixture** : si le champ custom est sur une `*Translation`, la factory doit setter `currentLocale` + `fallbackLocale` pour chaque locale **avant** d'écrire la valeur — sinon Doctrine persiste sur une translation aléatoire.

## Déroulement

### 1 — Cadrer le besoin

Demander (ou confirmer si déjà fourni) :

- **Type d'opération** :
  - **A** — modifier la **suite par défaut** (currencies, locales, channels, shipping/payment methods, products…) sans toucher au code PHP.
  - **B** — exposer un **champ custom** d'un modèle étendu dans une fixture (nécessite Factory + Fixture + service config).
  - **C** — créer une **suite custom** entièrement nouvelle (ex. `dev`, `demo`, `test`) qui charge un sous-ensemble ciblé.
- Pour le cas **B** : nom du modèle, nom du champ, type (scalaire / translatable / relation), valeur par défaut.

### 2 — Lister les fixtures dispo (toujours utile)

```bash
bin/console sylius:fixtures:list
```

Donne la liste des fixtures **réellement enregistrées** dans le container (vendor + custom). C'est la source de vérité pour savoir quel `key` utiliser sous `sylius_fixtures.suites.<suite>.fixtures.<key>`. Si ta fixture custom n'apparaît pas → tag `sylius_fixtures.fixture` manquant ou autowiring qui shadow.

### 3A — Modifier la suite par défaut (cas le plus fréquent)

Créer/éditer `config/packages/sylius_fixtures.yaml` :

```yaml
sylius_fixtures:
    suites:
        default: # Suite chargée par `sylius:fixtures:load`
            fixtures:
                currency:
                    options:
                        currencies: ['CZK', 'HUF']
                channel:
                    options:
                        custom:
                            cz_web_store:
                                name: "CZ Web Store"
                                code: "CZ_WEB"
                                locales:
                                    - "%locale%"
                                currencies:
                                    - "CZK"
                                enabled: true
                                hostname: "localhost"
                shipping_method:
                    options:
                        custom:
                            ups_eu:
                                code: "ups_eu"
                                name: "UPS_eu"
                                enabled: true
                                channels:
                                    - "CZ_WEB"
                payment_method:
                    options:
                        custom:
                            bank_transfer:
                                code: "bank_transfer_eu"
                                name: "Bank transfer_eu"
                                channels:
                                    - "CZ_WEB"
                                enabled: true
```

Points de vigilance :

- **`channels:` dans `shipping_method` / `payment_method`** doit référencer un **`code` de channel existant** (vendor `WEB-US`, `WEB-EU`, ou ton custom `CZ_WEB`). Une typo ne lève pas d'erreur explicite — la méthode est créée sans channel.
- **`currencies:`** sous `channel` doit lister des codes ISO **déjà déclarés** sous la fixture `currency`. Ordre des fixtures (currency avant channel) est garanti par les priorités vendor — ne pas inverser.
- **`%locale%`** est le paramètre Symfony app → résout en `en_US` par défaut, à overrider via `framework.default_locale`.

### 3B — Exposer un champ custom (modèle étendu)

Préalable : le champ doit déjà exister sur l'entité (ex. `ShippingMethod::deliveryConditions` ajouté via `/sylius:model`).

#### Étendre l'`ExampleFactory`

`src/Fixture/Factory/ShippingMethodExampleFactory.php` :

```php
<?php

declare(strict_types=1);

namespace App\Fixture\Factory;

use Sylius\Bundle\CoreBundle\Fixture\Factory\ShippingMethodExampleFactory as BaseShippingMethodExampleFactory;
use Sylius\Component\Core\Model\ShippingMethodInterface;
use Sylius\Component\Locale\Model\LocaleInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

final class ShippingMethodExampleFactory extends BaseShippingMethodExampleFactory
{
    public function create(array $options = []): ShippingMethodInterface
    {
        $shippingMethod = parent::create($options);

        if (!isset($options['deliveryConditions'])) {
            return $shippingMethod;
        }

        foreach ($this->getLocalesFromRepository() as $localeCode) {
            $shippingMethod->setCurrentLocale($localeCode);
            $shippingMethod->setFallbackLocale($localeCode);
            $shippingMethod->setDeliveryConditions($options['deliveryConditions']);
        }

        return $shippingMethod;
    }

    protected function configureOptions(OptionsResolver $resolver): void
    {
        parent::configureOptions($resolver);

        $resolver
            ->setDefault('deliveryConditions', null)
            ->setAllowedTypes('deliveryConditions', ['null', 'string'])
        ;
    }

    private function getLocalesFromRepository(): iterable
    {
        /** @var LocaleInterface[] $locales */
        $locales = $this->localeRepository->findAll();
        foreach ($locales as $locale) {
            yield $locale->getCode();
        }
    }
}
```

#### Étendre la `Fixture`

`src/Fixture/ShippingMethodFixture.php` :

```php
<?php

declare(strict_types=1);

namespace App\Fixture;

use Sylius\Bundle\CoreBundle\Fixture\ShippingMethodFixture as BaseShippingMethodFixture;
use Symfony\Component\Config\Definition\Builder\ArrayNodeDefinition;

final class ShippingMethodFixture extends BaseShippingMethodFixture
{
    protected function configureResourceNode(ArrayNodeDefinition $resourceNode): void
    {
        parent::configureResourceNode($resourceNode);

        $resourceNode
            ->children()
                ->scalarNode('deliveryConditions')->end()
        ;
    }
}
```

`configureResourceNode` étend le **schéma de configuration YAML** : sans cette étape, déclarer `deliveryConditions:` dans `sylius_fixtures.yaml` lève `Unrecognized option`. Ne pas oublier `parent::configureResourceNode()` — sinon tu perds tous les nœuds vendor (`code`, `name`, `enabled`, `channels`…).

#### Déclarer les services

`config/services.yaml` :

```yaml
services:
    sylius.fixture.example_factory.shipping_method:
        class: App\Fixture\Factory\ShippingMethodExampleFactory
        arguments:
            - "@sylius.factory.shipping_method"
            - "@sylius.repository.zone"
            - "@sylius.repository.shipping_category"
            - "@sylius.repository.locale"
            - "@sylius.repository.channel"
            - "@sylius.repository.tax_category"
        public: true

    sylius.fixture.shipping_method:
        class: App\Fixture\ShippingMethodFixture
        arguments:
            - "@sylius.manager.shipping_method"
            - "@sylius.fixture.example_factory.shipping_method"
        tags:
            - { name: sylius_fixtures.fixture }
```

Sources d'arguments à respecter (sinon le constructeur du parent échoue) :

- **`ExampleFactory`** : copier la signature du constructeur du parent vendor (`vendor/sylius/sylius/src/Sylius/Bundle/CoreBundle/Fixture/Factory/<Model>ExampleFactory.php`). L'ordre compte.
- **`Fixture`** : généralement `[manager, exampleFactory]`, mais certains fixtures vendor injectent en plus un repository ou un EventDispatcher → vérifier `vendor/sylius/sylius/src/Sylius/Bundle/CoreBundle/Resources/config/services/fixtures.xml`.

Exclusion d'autowiring requise pour `src/Fixture/` (cf. règles fondamentales).

#### Utiliser le champ dans le YAML

```yaml
sylius_fixtures:
    suites:
        default:
            fixtures:
                shipping_method:
                    options:
                        custom:
                            geis:
                                code: "geis"
                                name: "Geis"
                                enabled: true
                                channels:
                                    - "CZ_WEB"
                                deliveryConditions: "3-5 days"
```

### 3C — Suite custom dédiée

Pour un seed alternatif (demo, tests E2E, environnement preview) :

```yaml
sylius_fixtures:
    suites:
        demo:
            listeners:
                orm_purger: ~ # Purge la DB avant chargement
            fixtures:
                currency:
                    options:
                        currencies: ['EUR']
                channel:
                    options:
                        custom:
                            demo_store:
                                code: "DEMO"
                                name: "Demo Store"
                                hostname: "demo.localhost"
                                locales: ["en_US"]
                                currencies: ["EUR"]
                                enabled: true
```

Chargement : `bin/console sylius:fixtures:load demo`. Sans `orm_purger`, la suite est cumulative — utile pour empiler des seeds, dangereux si tu attendais un état propre.

### 4 — Charger et vérifier

```bash
bin/console cache:clear
bin/console sylius:fixtures:list   # ta fixture custom doit apparaître
bin/console sylius:fixtures:load   # suite "default"
# ou
bin/console sylius:fixtures:load demo
```

Vérifs post-load :

- Channel créé : `bin/console doctrine:query:sql "SELECT code, name FROM sylius_channel"`.
- Shipping/payment methods rattachés : check `sylius_shipping_method_channels` / `sylius_payment_method_channels`.
- Champ custom persisté (cas B) : `SELECT id, code, delivery_conditions FROM sylius_shipping_method_translation`.

### 5 — Clôture

Afficher :

- **Fichiers créés/modifiés** : `config/packages/sylius_fixtures.yaml`, `config/services.yaml`, éventuellement `src/Fixture/Factory/<Model>ExampleFactory.php` + `src/Fixture/<Model>Fixture.php`.
- **Commande de chargement** à jouer (`sylius:fixtures:load [suite]`).
- **Ce qui reste** : éventuelle suite `test` pour les tests E2E/Behat, hooks de purge, exposition admin du champ custom (→ `/sylius:form`).

## Pièges fréquents

- **Fixture custom invisible dans `sylius:fixtures:list`** : tag `sylius_fixtures.fixture` manquant, ou autowiring `App\:` qui auto-configure et conflit. Vérifier `debug:container --tag=sylius_fixtures.fixture` puis l'exclusion `src/Fixture/` dans `services.yaml`.
- **`Unrecognized option "deliveryConditions" under "sylius_fixtures.suites.default.fixtures.shipping_method.options.custom.geis"`** : la `Fixture` n'a pas étendu `configureResourceNode`. Le YAML est validé par schéma — tout champ non déclaré explose au boot.
- **`parent::__construct()` qui lève `Too few arguments`** : sur Sylius < 2.0.8 les args constructeur sont `private` ; il faut redéclarer le constructeur complet avec **tous** les arguments du parent. Depuis 2.0.8, ils sont `protected` → un simple `parent::__construct(...func_get_args())` suffit.
- **Multiple definitions for the same service id `sylius.fixture.example_factory.shipping_method`** : autowiring `App\:` non exclu sur `src/Fixture/Factory/`. Le service custom est défini deux fois (manuellement + auto). Ajouter `Fixture` à `exclude:`.
- **Channel custom créé sans currencies/locales** : la fixture `currency`/`locale` n'a pas tourné avant `channel` parce qu'on a override `priority` à la main. Laisser les priorités vendor par défaut (currency/locale ont priorité plus haute → tournent avant).
- **Champ translatable persisté sur une seule locale** : la factory n'a pas bouclé sur les locales (pas de `setCurrentLocale`/`setFallbackLocale`). Résultat : `SELECT * FROM ..._translation WHERE locale = 'fr_FR'` est vide alors que `en_US` est OK.
- **`sylius:fixtures:load` qui ne purge pas** : sans listener `orm_purger`, la suite est cumulative. Re-load → rows dupliquées, contraintes uniques qui claquent. Soit ajouter `orm_purger`, soit `doctrine:database:drop --force && doctrine:database:create && doctrine:migrations:migrate -n` avant.
- **Custom store fixture sur Sylius < 1.12** : l'option `custom` n'existait pas — il fallait override la fixture entière. Vérifier `composer.lock` si tu hérites d'un projet legacy.
- **Modifier la fixture vendor directement** : ne jamais patcher `vendor/sylius/sylius/src/Sylius/Bundle/CoreBundle/Fixture/` — réécrasé au prochain `composer update`. Toujours étendre via `App\Fixture\`.

## Argument optionnel

`/sylius:fixtures default channel CZ_WEB` — ajoute un channel `CZ_WEB` dans la suite `default`, sans toucher au code PHP.

`/sylius:fixtures shipping_method deliveryConditions:string` — étend `ShippingMethodExampleFactory` + `ShippingMethodFixture` pour exposer un champ `deliveryConditions` (préalable : champ ajouté via `/sylius:model`).

`/sylius:fixtures demo` — crée une suite custom `demo` avec purge ORM.

`/sylius:fixtures` sans argument — demande le type d'opération (modifier `default`, exposer un champ custom, créer une suite dédiée) et déroule.

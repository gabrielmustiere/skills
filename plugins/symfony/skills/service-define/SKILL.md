---
name: service-define
description: Déclare un service Symfony/Sylius — autowiring, autoconfiguration, `resource`, privé/public, env (`#[When]`). Déclenche sur "créer service", "enregistrer service", "autowire", "services.yaml". Impose injection constructeur.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /service-define — Enregistrer un service Symfony

Tu ajoutes une nouvelle classe au container Symfony. Tu t'appuies sur la configuration par défaut (`_defaults` avec `autowire`, `autoconfigure`, `resource: '../src/'`) plutôt que de déclarer le service à la main, et tu ne touches à `services.yaml` que pour les cas qui sortent du pattern.

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine du projet.
2. Vérifier `symfony/framework-bundle` dans les dépendances.
   - Présent → OK, continuer.
   - Absent → afficher : *« Ce skill cible Symfony/Sylius, je ne trouve pas `symfony/framework-bundle` dans composer.json. On continue quand même ou on change d'approche ? »* et attendre la réponse.
3. Lire `config/services.yaml` pour repérer la configuration `_defaults` et la règle `App\: resource: '../src/'`.
4. Si `sylius/sylius` est présent → signaler en une ligne que Sylius câble ses propres services via des bundles (`Sylius\Bundle\*`) et que les services applicatifs passent quand même par `App\:`.

## Règles fondamentales

- **Autowire + autoconfigure activés globalement** : les deux sont dans `_defaults` de `services.yaml` depuis Symfony Flex. Ne pas les répéter service par service, ne pas les désactiver.
- **Services privés par défaut** : `public: false` est la valeur par défaut dans `_defaults`. Ne passer `public: true` que pour un service récupéré via `$container->get()` dans un test ou un code legacy — jamais par confort.
- **Chargement par `resource`** : toute classe dans `src/` est déjà un service candidat via le bloc `App\: resource: '../src/'`. Ne pas re-déclarer un service ligne par ligne dans `services.yaml` si la classe est couverte par le resource.
- **Exclusions de resource** : `DependencyInjection/`, `Entity/`, `Kernel.php` sont exclus par défaut. Les entités Doctrine, les DTO, les value objects ne sont **pas** des services — ne pas les forcer dans le container.
- **`declare(strict_types=1)` en tête** de chaque classe PHP, cohérent avec `references/stacks/symfony.md`.
- **Pas de suffixe `Service`** sur un nom de classe. Préférer le nom métier (`ProductCreator`, `InvoiceRenderer`, `CartTotalCalculator`). Un service est le comportement, pas une étiquette.
- **Pas d'appel direct au container** : `$this->container->get('foo')` ou `$container->get(Foo::class)` est un anti-pattern. Toujours passer par l'injection constructeur (cf. `/symfony:service-wire`).

## Structure standard d'un `services.yaml`

```yaml
# config/services.yaml
parameters:

services:
    _defaults:
        autowire: true      # injection auto par type-hint
        autoconfigure: true # tag auto selon l'interface implémentée

    App\:
        resource: '../src/'
        exclude:
            - '../src/DependencyInjection/'
            - '../src/Entity/'
            - '../src/Kernel.php'
```

Toute classe sous `src/` est alors auto-enregistrée comme service privé, avec autowiring et autoconfiguration actifs. **Rien à écrire en plus pour un service courant.**

## Pattern canonique : nouveau service applicatif

### 1 — Écrire la classe

```php
// src/Service/ProductCreator.php
declare(strict_types=1);

namespace App\Service;

use App\Dto\CreateProductRequest;
use App\Entity\Product;
use Doctrine\ORM\EntityManagerInterface;
use Psr\Log\LoggerInterface;

final class ProductCreator
{
    public function __construct(
        private readonly EntityManagerInterface $em,
        private readonly LoggerInterface $logger,
    ) {}

    public function create(CreateProductRequest $request): Product
    {
        $product = new Product($request->name, $request->priceCents);
        $this->em->persist($product);
        $this->em->flush();

        $this->logger->info('Product created', ['id' => $product->getId()]);

        return $product;
    }
}
```

Aucune modification de `services.yaml` n'est nécessaire : `App\Service\ProductCreator` est couvert par le resource `App\:` et autowiré.

### 2 — Injecter ailleurs

N'importe quelle classe du container (contrôleur, autre service, commande) peut typehinter `ProductCreator` dans son constructeur :

```php
public function __construct(
    private readonly ProductCreator $creator,
) {}
```

### 3 — Vérifier

```bash
symfony console debug:container App\\Service\\ProductCreator
symfony console debug:autowiring ProductCreator
```

## Services publics (exception, pas la règle)

Un service doit être public **uniquement** si :

- Un test functional/unit l'extrait via `static::getContainer()->get(...)` — cas légitime.
- Un code legacy non refactorable l'appelle via `$container->get()` — à documenter.

```yaml
services:
    App\Service\LegacyBridge:
        public: true
```

Même en test, préférer `KernelTestCase::getContainer()->get(ProductCreator::class)` qui fonctionne avec les services privés depuis Symfony 4.1 (le container de test rend les services privés accessibles sans passer `public: true` en prod).

## Services par environnement

### Attribut `#[When]` / `#[WhenNot]` (Symfony 6.4+)

Registre le service **uniquement** dans les environnements cités :

```php
declare(strict_types=1);

namespace App\Service;

use Symfony\Component\DependencyInjection\Attribute\When;

#[When(env: 'dev')]
#[When(env: 'test')]
final class FakePaymentGateway implements PaymentGatewayInterface
{
    public function charge(int $cents): string
    {
        return 'fake_txn_'.bin2hex(random_bytes(8));
    }
}
```

Couplé à un alias d'interface, ça permet de substituer l'implémentation en dev/test sans toucher à la prod.

### Bloc `when@env` en YAML

```yaml
# config/services.yaml
when@dev:
    services:
        App\Service\DebugMailer:
            decorates: 'mailer.mailer'

when@prod:
    services:
        App\Service\StrictRateLimiter:
            arguments:
                $maxPerMinute: 60
```

Préférer l'attribut `#[When]` pour les classes entières, le bloc `when@env` pour des overrides d'arguments.

## Autoconfiguration (ce qui est auto-tagué)

Avec `autoconfigure: true`, une classe reçoit automatiquement des tags selon ce qu'elle implémente / étend :

| Ce que la classe fait | Tag reçu | Remarque |
|---|---|---|
| implémente `EventSubscriberInterface` | `kernel.event_subscriber` | pas besoin de tagger manuellement |
| étend `Command` | `console.command` | accessible via `bin/console` |
| implémente `VoterInterface` | `security.voter` | auto-enregistré |
| implémente `Twig\Extension\ExtensionInterface` | `twig.extension` | |
| attribut `#[AsMessageHandler]` | `messenger.message_handler` | |
| attribut `#[AsEventListener]` | `kernel.event_listener` | |
| attribut `#[AsCommand]` | `console.command` | remplace l'héritage de `Command` |

**Conséquence** : dans 90 % des cas on n'écrit **aucune ligne** de config — on implémente l'interface ou on pose l'attribut, point.

## Cas où on touche `services.yaml`

Ajouter une entrée manuelle uniquement si :

- **Arguments scalaires** à injecter (email, URL, clé API) → `/symfony:service-wire`.
- **Choix entre plusieurs implémentations** d'une même interface → `/symfony:service-wire` (bindings) ou alias + `#[Target]`.
- **Tag custom** non couvert par l'autoconfiguration → `/symfony:service-tags`.
- **Factory** pour instancier le service → `/symfony:service-tags`.
- **Public volontaire** (test, legacy) : `public: true`.

Sinon, la classe dans `src/` suffit.

## Pièges fréquents

- **Classe hors `src/`** (ex : `tests/`, `lib/`) → pas couverte par le resource. Ajouter un autre bloc `App\Tests\: resource: '../tests/'` uniquement si nécessaire, sinon enregistrer manuellement.
- **Classe finale vs non-finale** : les décorateurs (`#[AsDecorator]`, cf. `/symfony:service-tags`) nécessitent que la cible implémente une interface ou ne soit pas `final`. Par défaut, marquer les services `final` ; retirer le `final` uniquement si on est décoré.
- **DTO ou Value Object dans `src/`** → rentre dans le container et se fait autowirer, ce qui est absurde. Les mettre dans `src/Dto/` ou `src/Entity/` qui sont exclus, ou ajouter `exclude` pour le sous-dossier concerné.
- **Interface sans implémentation unique** : l'autowiring tombe en erreur (`Cannot autowire service X: argument $foo of method __construct() references interface Y but no such service exists`). Ajouter un alias (cf. `/symfony:service-wire`) ou une implémentation.
- **Cache container non rechargé** : après ajout d'une classe dans un dossier exclu ou modification du YAML, `symfony console cache:clear`.

## Déroulement

### 1 — Cadrer

- Rôle du service (verbe + objet métier : `ProductCreator`, `SendInvoiceMail`, `ResolveCartDiscount`).
- Dépendances déjà présentes dans le container (logger, EM, clients HTTP) vs scalaires à injecter (→ `/symfony:service-wire`).
- Portée : applicatif (`src/Service/`), domaine (`src/Domain/`), infra (`src/Infrastructure/`). La convention projet prime.

### 2 — Implémenter

- Créer la classe sous `src/...` avec `declare(strict_types=1)`, constructeur typehinté, méthode métier courte.
- **Ne pas toucher à `services.yaml`** sauf cas listé plus haut.
- Si l'implémentation varie par environnement, `#[When(env: 'dev')]` sur la variante dev + alias par défaut.

### 3 — Vérifier

```bash
symfony console debug:container App\\Service\\NomService
symfony console lint:container
vendor/bin/phpstan analyse src
```

`lint:container` valide la compilation (dépendances résolvables, arguments manquants détectés). Le lancer avant commit si on a touché la config.

### 4 — Clôture

Afficher :

- Fichier créé/modifié.
- Enregistrement effectif (resource auto ou entrée YAML).
- Tag(s) reçu(s) via autoconfiguration (si pertinent).
- Ce qui reste : câblage d'arguments scalaires (→ `/symfony:service-wire`), tags custom (→ `/symfony:service-tags`), test unitaire.

## Delta Sylius

- Les services applicatifs custom vivent sous `App\Service\...` comme dans n'importe quel projet Symfony.
- Pour étendre un service Sylius (`ProductVariantResolver`, `TaxCalculator`), utiliser la **décoration** (`#[AsDecorator('sylius.order_processor')]`, cf. `/symfony:service-tags`) — jamais de redéfinition totale du service vendor.
- Les services Sylius sont déjà publics (compatibilité bundle), mais les services `App\` restent privés.

## Argument optionnel

`/symfony:service-define ProductCreator` — crée le squelette de la classe sous `src/Service/` et vérifie l'enregistrement.

`/symfony:service-define src/Service/ExistingService.php` — audit d'un service existant (enregistrement, visibilité, autoconfiguration, respect des conventions).

`/symfony:service-define` sans argument — demande le nom et le rôle du service à l'utilisateur.

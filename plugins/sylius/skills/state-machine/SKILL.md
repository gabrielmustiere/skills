---
name: state-machine
description: Modifie une state machine Sylius 2.x (Symfony Workflow) — ajoute/retire un state/transition sur Order, Shipment, Payment ou Checkout, branche un listener `workflow.*.completed.*`. Piloter un order existant → `/sylius:order`.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /state-machine — Customiser une state machine Sylius

Tu aides à **modifier une state machine Sylius** (cycle de vie d'`Order`, `Shipment`, `Payment`, `ProductReview`, `OrderCheckout`, etc.). Depuis Sylius 2.x, les state machines s'appuient sur le **Symfony Workflow Component** (et non plus Winzou). Les graphes Sylius sont déclarés sous `framework.workflows.*` et chaque workflow a un nom unique via une constante `GRAPH` (ex. `Sylius\Component\Core\OrderCheckoutTransitions::GRAPH`).

Référence officielle : [docs.sylius.com/the-customization-guide/customizing-state-machines](https://docs.sylius.com/the-customization-guide/customizing-state-machines) · [State Machine Architecture](https://docs.sylius.com/the-book/architecture/state-machine) · [Symfony Workflow](https://symfony.com/doc/current/workflow.html) · [constantes de workflows vendor](https://github.com/Sylius/Sylius/tree/v2.0.7/src/Sylius/Bundle/CoreBundle/Resources/config/app/workflow).

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine.
2. Vérifier `sylius/sylius` **≥ 2.0**.
   - Absent → *« Ce skill cible Sylius 2.x (Symfony Workflow). Je ne trouve pas `sylius/sylius`. On continue quand même ? »*
3. Distinguer Symfony Workflow (2.x natif) vs **Winzou State Machine Bundle** (Sylius 1.x ou migration) :
   ```bash
   composer show | grep -E 'winzou/state-machine-bundle|symfony/workflow'
   ```
   - `symfony/workflow` installé → c'est le pipeline officiel, on continue.
   - Uniquement `winzou/state-machine-bundle` → l'app est sur le legacy ; rediriger vers [docs Sylius 1.14 state_machine](https://old-docs.sylius.com/en/1.14/customization/state_machine.html) et la [doc Winzou](https://github.com/winzou/StateMachineBundle). Les commandes `debug:config framework workflows` et les compiler passes sur `state_machine.<graph>.definition` ne s'appliquent pas.
4. Si la demande est *« déclencher un envoi d'e-mail quand l'order passe à `fulfilled` »* → le squelette est ici (event listener sur `workflow.sylius_order.completed.fulfill`), mais la construction du mail relève de **`/sylius:email`**. Préparer le pivot.
5. Si la demande est *« créer ou transiter un order par code »* (pas modifier la machine) → basculer sur **`/sylius:order`** : on y documente déjà `sm.factory`, `OrderCheckoutTransitions`, `OrderShippingTransitions`, `OrderPaymentTransitions`.
6. Si le besoin est *« réagir à une modification de données, pas à une transition »* (ex. un `preUpdate` Doctrine, un event Symfony applicatif) → rediriger vers **`/symfony:event-listen`** ou **`/symfony:event-subscribe`** : les workflow events de Symfony sont un sous-ensemble plus étroit que les events Doctrine/applicatifs.

## Règles fondamentales

- **Toujours référencer les workflows, states et transitions via les constantes**, jamais les chaînes brutes. `OrderCheckoutTransitions::GRAPH` au lieu de `'sylius_order_checkout'`, `OrderCheckoutStates::STATE_ADDRESSED` au lieu de `'addressed'`. Un typo de chaîne est silencieux et casse à l'exécution. Les constantes sont dans `vendor/sylius/sylius/src/Sylius/Bundle/CoreBundle/Resources/config/app/workflow/` et `src/Sylius/Component/*/Model/*Transitions.php` / `*States.php`.
- **La config YAML fusionne additivement pour les places et les transitions top-level, mais elle n'étend PAS les `from` d'une transition existante**. Ajouter un nouveau `place` ou une nouvelle `transition` → OK via `config/packages/_sylius.yaml`. Ajouter un `from` state supplémentaire à une transition existante (ex. permettre `fail` depuis `completed` en plus des `from` natifs) → **compiler pass obligatoire** sur la définition du container. C'est la limitation documentée.
- **Retirer un state ou une transition = compiler pass**, pas de YAML. Le Workflow Component Symfony n'a pas d'API pour "exclure" une transition native. Il faut manipuler directement `state_machine.<graph>.definition` dans le container (arguments 0 = places, 1 = transitions) via un `CompilerPassInterface` enregistré dans `Kernel::build()`.
- **Les compiler passes doivent s'enregistrer dans `Kernel::build()`**, pas dans une extension de bundle, pas dans `services.yaml`. `$container->addCompilerPass(new MaCompilerPass());`. Sinon la passe ne tourne jamais.
- **L'ordre des compiler passes compte**. Si tu retires une transition ET que tu en ajoutes une nouvelle qui utilise un state retiré, le résultat est non-déterministe. Utiliser la `PassConfig::TYPE_BEFORE_OPTIMIZATION` (défaut) et enregistrer les passes dans le bon ordre dans `Kernel::build()`.
- **Les callbacks passent par `kernel.event_listener`, pas par la config workflow**. L'event a une forme précise : `workflow.<graph>.<état_de_l_event>.<transition_ou_place>`. Les familles utiles :
  - `workflow.<graph>.guard.<transition>` — avant une transition, peut la bloquer (`$event->setBlocked(true)`).
  - `workflow.<graph>.leave.<place>` — à la sortie d'un state.
  - `workflow.<graph>.transition.<transition>` — pendant la transition.
  - `workflow.<graph>.enter.<place>` — à l'entrée d'un state.
  - `workflow.<graph>.entered.<place>` — après l'entrée (persistence OK, safe pour dispatch d'events métier).
  - `workflow.<graph>.completed.<transition>` — après succès complet de la transition (le plus courant pour les side-effects : mail, notification, flush secondaire).
  - `workflow.<graph>.announce.<transition>` — annonce les prochaines transitions dispo.
- **Override d'un listener Sylius = redéfinir le service avec la même clé**. Ex. `sylius.listener.workflow.order_checkout.resolve_order_shipping_state` → redéfinir ce service exact dans `config/services.yaml`. Pas besoin de compiler pass, pas besoin de decorator — la redéfinition écrase le vendor au moment du compile du container. Garder le même `event` dans le tag, sinon l'ancien listener peut rester câblé.
- **Logique métier lourde dans le listener, pas dans un guard**. Les guards doivent rester rapides (autorisation simple, vérif stock) — ils sont appelés à chaque fois que Sylius demande les transitions dispo (rendu Twig, API), pas juste à l'exécution. Une requête SQL dans un guard = N+1 garantis sur les pages où plusieurs orders sont listés.
- **Ne jamais modifier un fichier sous `vendor/sylius/sylius/src/Sylius/Bundle/*/Resources/config/app/workflow/`**. Toute customisation passe par `config/packages/_sylius.yaml` (ajout) ou compiler pass (suppression / extension de transition).
- **Cache obligatoire** après chaque modif de workflow YAML ou de compiler pass. `php bin/console cache:clear`. Sinon l'ancien graphe persiste dans le container compilé.

## Déroulement

### 1 — Cadrer le besoin

Demander (ou confirmer si fourni) :

- **Workflow cible** : `sylius_order`, `sylius_order_checkout`, `sylius_order_shipping`, `sylius_order_payment`, `sylius_payment`, `sylius_shipment`, `sylius_product_review`, ou un workflow custom déjà ajouté par l'app.
- **Type de customisation** :
  - *Ajouter* un state (place) et/ou une transition.
  - *Retirer* une transition native (ex. `skip_shipping`) ou un state (ex. `shipping_skipped`).
  - *Étendre* une transition existante en ajoutant un `from` supplémentaire.
  - *Réagir* à une transition (listener).
  - *Réécrire* le comportement d'un listener Sylius natif.
- **Déclencheur métier** : qui change l'état ? Utilisateur via UI, API, cron, event Doctrine, etc. (utile pour savoir où se trouve l'appel `apply()`).

### 2 — Repérer le graph et ses constantes

```bash
# Liste tous les workflows Sylius enregistrés
php bin/console debug:config framework workflows | grep sylius_

# Détail d'un workflow précis
php bin/console debug:config framework workflows sylius_order_checkout

# Visualiser en SVG (nécessite Graphviz / dot)
php bin/console workflow:dump sylius_order_checkout | dot -Tsvg -o checkout.svg
```

Constantes Sylius les plus utilisées :

| Workflow | Graph constant | States | Transitions |
|----------|---------------|--------|-------------|
| Order | `OrderTransitions::GRAPH` (`sylius_order`) | `OrderStates::STATE_CART`, `STATE_NEW`, `STATE_FULFILLED`, `STATE_CANCELLED` | `TRANSITION_CREATE`, `TRANSITION_FULFILL`, `TRANSITION_CANCEL` |
| Checkout | `OrderCheckoutTransitions::GRAPH` (`sylius_order_checkout`) | `STATE_CART`, `STATE_ADDRESSED`, `STATE_SHIPPING_SELECTED`, `STATE_SHIPPING_SKIPPED`, `STATE_PAYMENT_SELECTED`, `STATE_PAYMENT_SKIPPED`, `STATE_COMPLETED` | `TRANSITION_ADDRESS`, `TRANSITION_SELECT_SHIPPING`, `TRANSITION_SKIP_SHIPPING`, `TRANSITION_SELECT_PAYMENT`, `TRANSITION_SKIP_PAYMENT`, `TRANSITION_COMPLETE` |
| Order shipping | `OrderShippingTransitions::GRAPH` (`sylius_order_shipping`) | `STATE_CART`, `STATE_READY`, `STATE_SHIPPED`, `STATE_PARTIALLY_SHIPPED`, `STATE_CANCELLED` | `TRANSITION_REQUEST_SHIPPING`, `TRANSITION_SHIP`, `TRANSITION_CANCEL` |
| Order payment | `OrderPaymentTransitions::GRAPH` (`sylius_order_payment`) | `STATE_CART`, `STATE_AWAITING_PAYMENT`, `STATE_PARTIALLY_PAID`, `STATE_PAID`, `STATE_PARTIALLY_REFUNDED`, `STATE_REFUNDED`, `STATE_CANCELLED` | `TRANSITION_REQUEST_PAYMENT`, `TRANSITION_PAY`, `TRANSITION_PARTIALLY_PAY`, `TRANSITION_REFUND`, `TRANSITION_PARTIALLY_REFUND`, `TRANSITION_CANCEL` |
| Payment | `PaymentTransitions::GRAPH` (`sylius_payment`) | `STATE_CART`, `STATE_NEW`, `STATE_PROCESSING`, `STATE_AUTHORIZED`, `STATE_COMPLETED`, `STATE_FAILED`, `STATE_CANCELLED`, `STATE_REFUNDED` | `TRANSITION_CREATE`, `TRANSITION_PROCESS`, `TRANSITION_AUTHORIZE`, `TRANSITION_COMPLETE`, `TRANSITION_FAIL`, `TRANSITION_CANCEL`, `TRANSITION_REFUND` |
| Shipment | `ShipmentTransitions::GRAPH` (`sylius_shipment`) | `STATE_CART`, `STATE_READY`, `STATE_SHIPPED`, `STATE_CANCELLED` | `TRANSITION_CREATE`, `TRANSITION_SHIP`, `TRANSITION_CANCEL` |

### 3 — Cas A : Ajouter un state et une transition

Exemple : ajouter un state `custom_state` au checkout et y transiter depuis `addressed`.

```yaml
# config/packages/_sylius.yaml

framework:
    workflows:
        !php/const Sylius\Component\Core\OrderCheckoutTransitions::GRAPH:
            places:
                - 'custom_state'
            transitions:
                custom_transition:
                    from: !php/const Sylius\Component\Core\OrderCheckoutStates::STATE_ADDRESSED
                    to: 'custom_state'
```

Notes :

- Le `!php/const` de Symfony requiert `yaml.enable_php_constants` (activé par défaut dans Symfony 5+).
- `places` s'**ajoutent** à ceux natifs (pas de remplacement).
- `transitions` s'**ajoutent** si la clé (`custom_transition`) est nouvelle. Si tu reprends le nom d'une transition native avec une `from` différente, **la dernière déclaration gagne** et tu perds la native — utiliser alors le compiler pass de la section 5.
- Si tu veux qu'elle soit appliquée automatiquement, câbler un listener sur l'événement qui doit la déclencher et appeler `$stateMachine->apply($order, 'custom_transition')` (service `sm.factory` ou `Sylius\Component\Resource\StateMachine\StateMachineInterface`).

Rebuild :

```bash
php bin/console cache:clear
php bin/console debug:config framework workflows sylius_order_checkout   # vérifier la présence du state + transition
```

### 4 — Cas B : Retirer une transition ou un state

#### 4.1 — Retirer une transition (ex. `skip_shipping`)

```php
<?php
// src/DependencyInjection/Compiler/RemoveSkipShippingTransitionCompilerPass.php

namespace App\DependencyInjection\Compiler;

use Sylius\Component\Core\OrderCheckoutTransitions;
use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;

final class RemoveSkipShippingTransitionCompilerPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container): void
    {
        $graph = OrderCheckoutTransitions::GRAPH;
        $transitionToRemove = OrderCheckoutTransitions::TRANSITION_SKIP_SHIPPING;

        $definitionId = sprintf('state_machine.%s.definition', $graph);
        if (!$container->hasDefinition($definitionId)) {
            return;
        }

        $definition = $container->getDefinition($definitionId);
        $transitions = $definition->getArgument(1);

        foreach ($transitions as $i => $ref) {
            $transitionDef = $container->getDefinition((string) $ref);
            if ($transitionDef->getArgument(0) === $transitionToRemove) {
                unset($transitions[$i]);
                break;
            }
        }

        $definition->replaceArgument(1, array_values($transitions));
    }
}
```

#### 4.2 — Retirer un state (ex. `shipping_skipped`)

Retirer un state impose aussi de **retirer toutes les transitions qui pointent vers lui ou en partent** — sinon le graphe est inconsistant et le compile échoue.

```php
<?php
// src/DependencyInjection/Compiler/RemoveShippingSkippedStateCompilerPass.php

namespace App\DependencyInjection\Compiler;

use Sylius\Component\Core\OrderCheckoutStates;
use Sylius\Component\Core\OrderCheckoutTransitions;
use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;

final class RemoveShippingSkippedStateCompilerPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container): void
    {
        $graph = OrderCheckoutTransitions::GRAPH;
        $stateToRemove = OrderCheckoutStates::STATE_SHIPPING_SKIPPED;

        $definitionId = sprintf('state_machine.%s.definition', $graph);
        if (!$container->hasDefinition($definitionId)) {
            return;
        }

        $definition = $container->getDefinition($definitionId);
        $places = $definition->getArgument(0);
        $transitions = $definition->getArgument(1);

        $placeKey = array_search($stateToRemove, $places, true);
        if ($placeKey !== false) {
            unset($places[$placeKey]);
            $places = array_values($places);
        }

        foreach ($transitions as $i => $ref) {
            $transitionDef = $container->getDefinition((string) $ref);
            $from = (array) $transitionDef->getArgument(1);
            $to = (array) $transitionDef->getArgument(2);

            if (in_array($stateToRemove, $from, true) || in_array($stateToRemove, $to, true)) {
                unset($transitions[$i]);
            }
        }

        $definition->replaceArgument(0, $places);
        $definition->replaceArgument(1, array_values($transitions));
    }
}
```

#### 4.3 — Enregistrer les compiler passes

```php
// src/Kernel.php

use App\DependencyInjection\Compiler\RemoveShippingSkippedStateCompilerPass;
use App\DependencyInjection\Compiler\RemoveSkipShippingTransitionCompilerPass;
use Symfony\Component\DependencyInjection\ContainerBuilder;

protected function build(ContainerBuilder $container): void
{
    parent::build($container);

    $container->addCompilerPass(new RemoveSkipShippingTransitionCompilerPass());
    $container->addCompilerPass(new RemoveShippingSkippedStateCompilerPass());
}
```

Rebuild + contrôle :

```bash
php bin/console cache:clear
php bin/console debug:config framework workflows sylius_order_checkout   # skip_shipping / shipping_skipped absents
```

### 5 — Cas C : Étendre une transition existante (ajouter des `from`)

La config YAML classique ne peut **pas** ajouter un `from` state supplémentaire à une transition native. Exemple : autoriser `fail` sur `sylius_payment` depuis `completed` (en plus du `processing` natif). Passage obligatoire par un compiler pass qui injecte une nouvelle `Definition` de `Transition` si la combinaison `name:from:to` n'existe pas encore.

```php
<?php
// src/DependencyInjection/Compiler/ExtendPaymentWorkflowTransitionPass.php

namespace App\DependencyInjection\Compiler;

use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Definition;
use Symfony\Component\Workflow\Transition;

final class ExtendPaymentWorkflowTransitionPass implements CompilerPassInterface
{
    private const REQUIRED_TRANSITIONS = [
        ['name' => 'process', 'from' => 'authorized', 'to' => 'processing'],
        ['name' => 'fail',    'from' => 'completed',  'to' => 'failed'],
    ];

    public function process(ContainerBuilder $container): void
    {
        if (!$container->hasDefinition('state_machine.sylius_payment.definition')) {
            return;
        }

        $definition = $container->getDefinition('state_machine.sylius_payment.definition');
        $transitions = $definition->getArgument(1);

        $existing = $this->extractExistingTransitions($container, $transitions);

        foreach (self::REQUIRED_TRANSITIONS as $t) {
            $key = sprintf('%s:%s:%s', $t['name'], $t['from'], $t['to']);
            if (in_array($key, $existing, true)) {
                continue;
            }

            $transition = new Definition(Transition::class);
            $transition->setArguments([$t['name'], [$t['from']], [$t['to']]]);
            $transitions[] = $transition;
        }

        $definition->setArgument(1, array_values($transitions));
    }

    /**
     * @param array<Definition|\Symfony\Component\DependencyInjection\Reference> $transitions
     * @return array<string>
     */
    private function extractExistingTransitions(ContainerBuilder $container, array $transitions): array
    {
        $keys = [];

        foreach ($transitions as $transition) {
            if (!$transition instanceof Definition) {
                $transition = $container->getDefinition((string) $transition);
            }

            $args = $transition->getArguments();
            if (count($args) < 3) {
                continue;
            }

            [$name, $froms, $tos] = $args;
            foreach ((array) $froms as $from) {
                foreach ((array) $tos as $to) {
                    $keys[] = sprintf('%s:%s:%s', $name, $from, $to);
                }
            }
        }

        return $keys;
    }
}
```

Enregistrement dans `Kernel::build()` comme en §4.3.

### 6 — Cas D : Ajouter un callback (event listener)

Exemple : envoyer un e-mail quand le checkout se termine (transition `complete`).

```php
<?php
// src/EventListener/Workflow/OrderCheckout/SendGiftCodeAfterCompletionListener.php

namespace App\EventListener\Workflow\OrderCheckout;

use Symfony\Component\Workflow\Event\CompletedEvent;

final class SendGiftCodeAfterCompletionListener
{
    public function __invoke(CompletedEvent $event): void
    {
        $order = $event->getSubject();
        // dispatch e-mail ici — voir /sylius:email pour l'intégration Sender
    }
}
```

```yaml
# config/services.yaml

services:
    app.listener.workflow.order_checkout.send_gift_code:
        class: App\EventListener\Workflow\OrderCheckout\SendGiftCodeAfterCompletionListener
        tags:
            - { name: kernel.event_listener, event: 'workflow.sylius_order_checkout.completed.complete', priority: 100 }
```

Choix de l'événement selon ce qu'on veut faire :

- Bloquer la transition → `workflow.<graph>.guard.<transition>` + `$event->setBlocked(true, 'raison')`.
- Effet de bord après persistance → `workflow.<graph>.completed.<transition>` (le plus courant).
- Effet de bord à l'entrée d'un state → `workflow.<graph>.entered.<place>`.

Signatures d'événements utiles :

| Event | Classe | Quand |
|-------|--------|-------|
| `guard.<transition>` | `GuardEvent` | Avant — peut bloquer via `setBlocked()` |
| `leave.<place>` | `LeaveEvent` | À la sortie d'un state |
| `transition.<transition>` | `TransitionEvent` | Pendant la transition (avant l'update du state) |
| `enter.<place>` | `EnterEvent` | Entrée d'un state (avant persist) |
| `entered.<place>` | `EnteredEvent` | Après persist (safe pour dispatch d'events métier) |
| `completed.<transition>` | `CompletedEvent` | Après la transition complète |
| `announce.<transition>` | `AnnounceEvent` | Annonce les prochaines transitions dispo |

### 7 — Cas E : Override d'un listener Sylius natif

Redéfinir le **même service ID** que celui du vendor. Ex. customiser le resolver du shipping state après complétion du checkout :

```yaml
# config/services.yaml

services:
    sylius.listener.workflow.order_checkout.resolve_order_shipping_state:
        class: App\EventListener\Workflow\OrderCheckout\ResolveOrderShippingStateListener
        tags:
            - { name: kernel.event_listener, event: 'workflow.sylius_order_checkout.completed.complete', priority: 100 }
```

```bash
# Vérifier qui est câblé sur l'événement
php bin/console debug:event workflow.sylius_order_checkout.completed.complete

# Confirmer la classe prise en compte
php bin/console debug:container sylius.listener.workflow.order_checkout.resolve_order_shipping_state
```

### 8 — Debug et vérification

```bash
# Workflows disponibles
php bin/console debug:config framework workflows | grep sylius_

# Détail d'un workflow
php bin/console debug:config framework workflows sylius_order_checkout

# Listeners câblés sur une transition
php bin/console debug:event workflow.sylius_order_checkout.completed.complete

# Vue graphique (Graphviz requis)
php bin/console workflow:dump sylius_order_checkout | dot -Tsvg -o checkout.svg

# Inspecter la définition directement
php bin/console debug:container state_machine.sylius_order_checkout.definition
```

À faire après chaque modif :

```bash
php bin/console cache:clear
```

### 9 — Cas legacy : Winzou State Machine Bundle

Sur un projet Sylius 1.x ou migré incomplet, la state machine peut encore être pilotée par [winzou/state-machine-bundle](https://github.com/winzou/StateMachineBundle). Signes : config sous `winzou_state_machine:` (pas `framework.workflows:`), callbacks sous `callbacks.before/after` (pas des listeners Symfony events), services `sm.factory` qui renvoient `WinzouStateMachine` et pas une instance de `Symfony\Component\Workflow\Workflow`.

Dans ce cas :

- Les compiler passes sur `state_machine.<graph>.definition` ne s'appliquent pas.
- Ajouter une transition ou un state se fait en YAML sous `winzou_state_machine.<graph>.transitions` / `.states`.
- Les callbacks se déclarent sous `winzou_state_machine.<graph>.callbacks.(before|after)`.
- Référence : [docs Sylius 1.14 state_machine](https://old-docs.sylius.com/en/1.14/customization/state_machine.html).
- Si migration 2.x prévue, prévoir la coexistence : Sylius 2.0 a basculé sur Symfony Workflow pour les workflows natifs Core, mais Winzou peut rester installé pour du code applicatif legacy. Distinguer le graphe par le bundle qui le fournit.

### 10 — Clôture

Afficher :

- **Fichiers créés/modifiés** :
  - `config/packages/_sylius.yaml` (ajout places/transitions via YAML)
  - `src/DependencyInjection/Compiler/<Nom>CompilerPass.php` (suppression ou extension de transition)
  - `src/Kernel.php` (méthode `build()` avec `addCompilerPass()`)
  - `src/EventListener/Workflow/<Graph>/<Nom>Listener.php` (callbacks)
  - `config/services.yaml` (tag `kernel.event_listener` et/ou override d'un service Sylius)
- **Commandes** : `php bin/console cache:clear`, `php bin/console debug:config framework workflows <graph>`, `php bin/console debug:event workflow.<graph>.completed.<transition>`.
- **Ce qui reste à faire** : traduction des messages flash liés aux transitions (clés `sylius.order.transition.*` dans `translations/flashes.<locale>.yaml`), tests fonctionnels des nouveaux paths (Behat ou PHPUnit), audit des consommateurs de l'ancien state (rapports BI, dashboards, exports) qui pourraient ignorer un nouveau state `custom_state`.

## Pièges fréquents

- **Chaîne brute au lieu de constante** : `'sylius_order_checkout'` fonctionne mais se désynchronise silencieusement si Sylius renomme le graph dans une version future. Toujours `OrderCheckoutTransitions::GRAPH`.
- **YAML qui "redéfinit" une transition native** : donner à une nouvelle transition le **même nom** qu'une native en changeant le `from` — la native disparaît sans warning, tous les appels `apply($order, 'address')` qui s'appuyaient sur l'ancien `from` échouent en runtime. Utiliser un nom distinct (`custom_address`) ou passer par un compiler pass d'extension (§5).
- **Compiler pass oubliée dans `Kernel::build()`** : instanciée correctement ailleurs mais jamais enregistrée → aucune action, pas d'erreur. Toujours vérifier `debug:config framework workflows <graph>` après `cache:clear`.
- **Suppression d'un state sans nettoyer ses transitions** : le `WorkflowDumper` ou le compile du container peut lever `InvalidDefinitionException: The state "shipping_skipped" is not defined` à cause d'une transition orpheline. La passe §4.2 doit balayer **toutes** les transitions et retirer celles qui pointent vers ou partent du state.
- **Guard qui lève une exception** : au lieu de `$event->setBlocked(true, 'raison')`, le code lève → toute la liste des transitions dispo (appelée en contexte UI) explose. Un guard ne doit **pas** throw, juste setBlocked.
- **Event `completed` utilisé pour modifier le subject** : la transition est déjà persistée ; modifier l'entité dans `CompletedEvent` nécessite un flush explicite et peut re-déclencher un workflow si l'update change un state. Préférer `EnteredEvent` pour modifier et `CompletedEvent` pour dispatcher (mail, queue, webhook).
- **Override d'un listener Sylius sans garder le même tag `event`** : le service est redéfini avec la bonne classe, mais le nouveau tag pointe sur un événement voisin → l'ancien listener Sylius reste câblé sur l'événement d'origine, le nouveau ne se déclenche pas. Toujours aligner `event` sur celui du vendor.
- **`php/const` non interprété** : sur Symfony < 5.1 ou avec `yaml.enable_php_constants: false`, `!php/const ...` est lu comme une chaîne `"!php/const ..."` et casse le graphe silencieusement. Vérifier la version, ou écrire en XML.
- **Listener déclaré en tant que `EventSubscriber` mais appelé comme listener invokable** : `__invoke()` ne suffit pas si le service a `kernel.event_subscriber` au lieu de `kernel.event_listener` → la méthode `getSubscribedEvents()` est requise. Choisir un pattern et s'y tenir.
- **Compiler pass qui mute la définition sans `array_values()`** : `unset($transitions[$i])` laisse des trous d'index, certaines versions Symfony lèvent à la compilation. Toujours `$definition->replaceArgument(1, array_values($transitions))`.
- **`debug:event` ne liste plus rien après override** : signe que le service redéfini a perdu le tag `kernel.event_listener`. Recopier explicitement le tag complet (nom, event, priority) en redéfinissant.
- **Migration 1.x → 2.x avec winzou ET workflow cohabitants** : les callbacks winzou continuent à tourner pour les graphes qu'il gère, mais les nouveaux graphes ne sont visibles que via `debug:config framework workflows`, pas via `debug:config winzou_state_machine`. Faire l'inventaire graphe par graphe avant de migrer les callbacks.

## Argument optionnel

`/sylius:state-machine add sylius_order_checkout custom_state --from=addressed` — scaffold un ajout de place + transition depuis `STATE_ADDRESSED` via `config/packages/_sylius.yaml`.

`/sylius:state-machine remove-transition sylius_order_checkout skip_shipping` — génère le compiler pass qui retire la transition `skip_shipping` et l'enregistre dans `Kernel::build()`.

`/sylius:state-machine remove-state sylius_order_checkout shipping_skipped` — idem pour un state, avec nettoyage des transitions associées.

`/sylius:state-machine extend-transition sylius_payment fail --from=completed --to=failed` — ajoute un `from` supplémentaire à une transition existante via compiler pass (cas où le YAML ne suffit pas).

`/sylius:state-machine listen sylius_order_checkout complete --on=completed` — scaffold un listener sur `workflow.sylius_order_checkout.completed.complete` + tag dans `config/services.yaml`.

`/sylius:state-machine override sylius.listener.workflow.order_checkout.resolve_order_shipping_state` — scaffold la redéfinition d'un service listener Sylius natif en conservant event + priorité.

`/sylius:state-machine` sans argument — demande le graph cible, le type d'action (add / remove-transition / remove-state / extend-transition / listen / override), et guide pas à pas (YAML ou compiler pass → enregistrement `Kernel::build()` → cache:clear → vérif `debug:config`).

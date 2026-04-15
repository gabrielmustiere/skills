---
name: menu
description: Customise un menu Sylius 2.x : sidebar admin et menu compte shop via listener sur `MenuBuilderEvent` (`sylius.menu.admin.main`, `sylius.menu.shop.account`), boutons et onglets de form via Twig Hooks, icônes Tabler.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /menu — Customiser un menu Sylius

Tu aides à **ajouter, réordonner ou retirer des entrées** dans les menus Sylius 2.x. La règle de base : **Sylius 2.x utilise deux mécanismes selon le type de menu** — Event Listener pour les menus *structurels* (sidebar admin, navigation compte client), Twig Hook pour les menus *d'actions* (boutons en haut à droite des pages de détail) et les onglets de formulaires. Choisir le mauvais mécanisme = le code ne s'exécute jamais ou s'enregistre dans un event déprécié.

Référence officielle : [docs.sylius.com/the-customization-guide/customizing-menus](https://docs.sylius.com/the-customization-guide/customizing-menus).

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine.
2. Vérifier `sylius/sylius` dans les dépendances.
   - Présent → OK.
   - Absent → *« Ce skill cible Sylius 2.x (menus via MenuBuilderEvent + Twig Hooks). Je ne trouve pas `sylius/sylius`. On continue quand même ? »*
3. Vérifier la version Sylius. Sur 1.x, les action menus passent par des events dépréciés (ex. `sylius.menu.admin.order.show`) — si le projet n'est pas migré, basculer vers `/sylius:template` pour la partie hooks, ou documenter le pattern legacy séparément.
4. Si l'objectif est un ajout purement visuel dans un panneau existant (ex. un badge à côté d'un item) → c'est un cas **`/sylius:template`** (Twig Hook sur le template du menu), pas un ajout d'entrée.

## Règles fondamentales

- **Deux mécanismes, deux usages** :
  - **Event Listener** (`MenuBuilderEvent`) → menus structurels : sidebar admin (`sylius.menu.admin.main`), sidebar compte client (`sylius.menu.shop.account`). Arborescence manipulée via l'API Knp Menu (`$menu->addChild()`, `setLabel()`, `setLabelAttribute('icon', ...)`).
  - **Twig Hook** → tout le reste : boutons d'action des pages de détail (`sylius_admin.<resource>.show.content.header.title_block.actions`), onglets des formulaires produit/variante (`*.content.form.side_navigation` + `*.content.form.sections`).
- **Icônes = Tabler + `ux_icon()`.** Sylius 2.x a abandonné Semantic UI. Les icônes viennent du set [Tabler](https://tabler-icons.io/) (`tabler:plus`, `tabler:check`, `tabler:star`, etc.) rendues par `{{ ux_icon('tabler:xxx', {class: 'icon'}) }}` en Twig ou `setLabelAttribute('icon', 'tabler:xxx')` côté Knp Menu.
- **`setExtra('always_open', true)` sur le parent, pas l'enfant.** Pour qu'une catégorie custom soit dépliée au chargement, l'attribut va sur le nœud parent (celui qui a des enfants), jamais sur un item feuille.
- **Pair `side_navigation` + `sections` pour un onglet.** Un onglet custom dans le form produit demande **deux hookables** configurés ensemble : un pour la vignette cliquable (`form.side_navigation`), un pour le contenu (`form.sections`). Les `data-bs-target` et l'`id` doivent matcher exactement (`#product-<nom>` par convention).
- **`create` ET `update` en parallèle.** Sur les forms produit/variant, `new` et `edit` ont des hooks séparés — il faut déclarer les 4 hookables (`create.*.side_navigation`, `create.*.sections`, `update.*.side_navigation`, `update.*.sections`) en pointant sur les **mêmes templates**, sinon l'onglet n'apparaît qu'à la création ou qu'à l'édition.
- **Listeners autoconfigurés ? Non**. Contrairement aux form extensions, les listeners sur `MenuBuilderEvent` doivent être enregistrés **manuellement** dans `config/services.yaml` avec le tag `kernel.event_listener` (event + method). L'autoconfigure n'attrape pas ce tag spécifique.
- **`cache:clear` systématique** après modif d'un listener ou d'un YAML de hook.

## Déroulement

### 1 — Cadrer le besoin

Demander (ou confirmer) :

- **Type de menu** : sidebar admin, sidebar compte client, boutons d'action sur une page de détail (customer/order/…), onglets de form (product/variant), autre ?
- **Opération** : ajouter une entrée, ajouter une sous-entrée, réordonner, masquer une entrée native, modifier un label/icône ?
- **Contexte** : route/path concerné, icône souhaitée (Tabler), traduction des labels.

Le choix du mécanisme découle directement du type de menu — voir la table ci-dessous.

### 2 — Table de décision

| Type de menu | Path | Mécanisme | Event / Hook |
| --- | --- | --- | --- |
| Admin Main Menu (sidebar) | `/admin/` | **Event Listener** | `sylius.menu.admin.main` |
| Admin Customer Show Menu (actions) | `/admin/customers/{id}` | **Twig Hook** | `sylius_admin.customer.show.content.header.title_block.actions` |
| Admin Order Show Menu (actions) | `/admin/orders/{id}` | **Twig Hook** | `sylius_admin.order.show.content.header.title_block.actions` |
| Admin Product Form Tabs | `/admin/products/new` + `/{id}/edit` | **Twig Hook** (paire) | `sylius_admin.product.{create\|update}.content.form.{side_navigation\|sections}` |
| Admin Product Variant Form Tabs | `/admin/products/{pId}/variants/create` + `/{id}/edit` | **Twig Hook** (paire) | `sylius_admin.product_variant.{create\|update}.content.form.{side_navigation\|sections}` |
| Shop Account Menu (sidebar) | `/account/dashboard/` | **Event Listener** | `sylius.menu.shop.account` |

### 3 — Cas 1 : Event Listener (sidebar admin ou compte client)

#### Étape 1. Créer le listener

Exemple pour l'admin :

```php
<?php
# src/Menu/AdminMenuListener.php

namespace App\Menu;

use Sylius\Bundle\UiBundle\Menu\Event\MenuBuilderEvent;

final class AdminMenuListener
{
    public function addAdminMenuItems(MenuBuilderEvent $event): void
    {
        $menu = $event->getMenu();

        $newSubmenu = $menu
            ->addChild('new')
            ->setLabel('Custom Admin Menu')
            ->setExtra('always_open', true) // déplié au chargement
        ;

        $newSubmenu
            ->addChild('new-subitem', ['route' => 'app_admin_custom_index'])
            ->setLabel('Custom Admin Menu Item')
            ->setLabelAttribute('icon', 'tabler:star')
        ;
    }
}
```

Exemple pour le compte client :

```php
<?php
# src/Menu/AccountMenuListener.php

namespace App\Menu;

use Sylius\Bundle\UiBundle\Menu\Event\MenuBuilderEvent;

final class AccountMenuListener
{
    public function addAccountMenuItems(MenuBuilderEvent $event): void
    {
        $menu = $event->getMenu();

        $menu
            ->addChild('loyalty', ['route' => 'app_shop_account_loyalty'])
            ->setLabel('app.account.loyalty')
            ->setLabelAttribute('icon', 'tabler:star')
        ;
    }
}
```

#### Étape 2. Enregistrer le listener

```yaml
# config/services.yaml
services:
    app.listener.admin.menu_builder:
        class: App\Menu\AdminMenuListener
        tags:
            - { name: kernel.event_listener, event: sylius.menu.admin.main, method: addAdminMenuItems }

    app.listener.shop.menu_builder:
        class: App\Menu\AccountMenuListener
        tags:
            - { name: kernel.event_listener, event: sylius.menu.shop.account, method: addAccountMenuItems }
```

Points clés :

- Le `name` du tag est **`kernel.event_listener`** (standard Symfony), pas un tag Sylius-spécifique.
- La méthode indiquée (`method:`) doit exister sur la classe.
- L'ordre entre listeners peut être contrôlé via `priority: N` dans le tag (défaut : `0`, plus haut = exécuté avant).

#### Étape 3. Traductions

Les labels littéraux (`'Custom Admin Menu'`) sont affichés tels quels. Pour passer par le système de trad, utiliser une clé et déclarer la trad :

```yaml
# translations/messages.fr.yaml
app:
    account:
        loyalty: Mon programme fidélité
```

### 4 — Cas 2 : Twig Hook — boutons d'action sur page de détail

Pattern : ajouter un `<a>` ou un `<form>` dans le dropdown d'actions en haut à droite d'une page `show`.

#### Exemple — bouton "Créer un client" sur la page client admin

Étape 1. Config :

```yaml
# config/packages/_sylius.yaml
sylius_twig_hooks:
    hooks:
        'sylius_admin.customer.show.content.header.title_block.actions':
            create:
                template: 'admin/customer/show/content/header/title_block/actions/create.html.twig'
```

Étape 2. Template :

```twig
{# templates/admin/customer/show/content/header/title_block/actions/create.html.twig #}

<a href="{{ path('sylius_admin_customer_create') }}" class="dropdown-item">
    {{ ux_icon(action.icon|default('tabler:plus'), {'class': 'icon dropdown-item-icon'}) }}
    {{ 'sylius.ui.create'|trans }}
</a>
```

#### Exemple — bouton "Compléter le paiement" sur la page commande admin

```yaml
# config/packages/_sylius.yaml
sylius_twig_hooks:
    hooks:
        'sylius_admin.order.show.content.header.title_block.actions':
            complete_payment:
                template: 'admin/order/show/content/header/title_block/actions/complete_payment.html.twig'
```

```twig
{# templates/admin/order/show/content/header/title_block/actions/complete_payment.html.twig #}
{% set order = hookable_metadata.context.resource %}
{% set payment = order.getPayments.first %}

{% if sylius_sm_can(payment, constant('Sylius\\Component\\Payment\\PaymentTransitions::GRAPH'), constant('Sylius\\Component\\Payment\\PaymentTransitions::TRANSITION_COMPLETE')) %}
    <form action="{{ path('sylius_admin_order_payment_complete', {'orderId': order.id, 'id': payment.id}) }}" method="POST" novalidate>
        <input type="hidden" name="_method" value="PUT">
        <input type="hidden" name="_csrf_token" value="{{ csrf_token(payment.id) }}" />
        <button type="submit" class="dropdown-item" {{ sylius_test_html_attribute('complete-payment', payment.id) }}>
            {{ ux_icon('tabler:check', {'class': 'icon dropdown-item-icon'}) }}
            {{ 'sylius.ui.complete'|trans }}
        </button>
    </form>
{% endif %}
```

Points clés :

- `hookable_metadata.context.resource` expose l'entité en cours (`Customer`, `Order`, etc.).
- Les boutons utilisent la classe `dropdown-item` — ne pas sortir du style Tabler/Bootstrap sinon ils apparaissent décalés.
- Pour une action sensible (POST), toujours inclure CSRF + état autorisé par state machine (`sylius_sm_can`).

### 5 — Cas 3 : Twig Hook — onglets de formulaires (Product / ProductVariant)

Pattern : ajouter un onglet custom (ex. "Manufacturer") dans la navigation latérale du form produit, avec son contenu associé.

#### Étape 1. Config — déclarer les 4 hookables (create + update × side_navigation + sections)

```yaml
# config/packages/_sylius.yaml
sylius_twig_hooks:
    hooks:
        'sylius_admin.product.create.content.form.side_navigation':
            manufacturer:
                template: 'admin/product/form/side_navigation/manufacturer.html.twig'

        'sylius_admin.product.create.content.form.sections':
            manufacturer:
                template: 'admin/product/form/sections/manufacturer.html.twig'

        'sylius_admin.product.update.content.form.side_navigation':
            manufacturer:
                template: 'admin/product/form/side_navigation/manufacturer.html.twig'

        'sylius_admin.product.update.content.form.sections':
            manufacturer:
                template: 'admin/product/form/sections/manufacturer.html.twig'
```

#### Étape 2. Template du bouton d'onglet (`side_navigation`)

```twig
{# templates/admin/product/form/side_navigation/manufacturer.html.twig #}

<button
    type="button"
    class="list-group-item list-group-item-action {% if hookable_metadata.configuration.active|default(false) %}active{% endif %}"
    data-bs-toggle="tab"
    data-bs-target="#product-manufacturer"
    role="tab"
>
    {{ 'app.product.manufacturer'|trans }}
</button>
```

#### Étape 3. Template du contenu (`sections`)

```twig
{# templates/admin/product/form/sections/manufacturer.html.twig #}

<div class="tab-pane {% if hookable_metadata.configuration.active|default(false) %}show active{% endif %}" id="product-manufacturer" role="tabpanel" tabindex="0">
    <div class="card mb-3">
        <div class="card-header">
            <div class="card-title">
                {{ 'app.product.manufacturer'|trans }}
            </div>
        </div>
        <div class="card-body">
            <div class="tab-content">
                {# afficher ici les champs ajoutés via /sylius:form sur ProductType #}
                {{ form_row(form.manufacturer) }}
            </div>
        </div>
    </div>
</div>
```

Points clés :

- `data-bs-target="#product-manufacturer"` **doit** matcher l'`id` de la `div.tab-pane`. Typo ou divergence = onglet vide.
- L'onglet n'est actif qu'au chargement si un autre hookable le marque (`configuration: { active: true }`) — sinon, reste masqué tant que l'utilisateur ne clique pas dessus.
- Pour insérer un form field custom : ajouter d'abord le champ via **`/sylius:form`** (extension de `ProductType`), puis `{{ form_row(form.<champ>) }}` ici.
- Même pattern pour `ProductVariant` : remplacer `product` par `product_variant` dans les noms de hooks.

### 6 — Opérations complémentaires

#### Réordonner un item de sidebar

Les items natifs Sylius dans `sylius.menu.admin.main` ont des positions définies par leur ordre de déclaration. Pour repositionner :

```php
public function addAdminMenuItems(MenuBuilderEvent $event): void
{
    $menu = $event->getMenu();

    // déplacer 'catalog' après 'customers'
    $catalog = $menu->getChild('catalog');
    $menu->removeChild('catalog');
    $menu->addChild($catalog); // réinséré en fin
}
```

L'API Knp Menu complète : [github.com/KnpLabs/KnpMenu](https://github.com/KnpLabs/KnpMenu/blob/master/doc/01-Basic-Menus.md).

#### Masquer une entrée native

```php
$menu->removeChild('configuration');
```

Attention : sur une mise à jour Sylius, si la clé change, le `removeChild` devient silencieusement inopérant. Vérifier au profiler (icône *Events* → listeners sur `sylius.menu.admin.main`) que la modif a bien été appliquée.

#### Désactiver un bouton d'action natif

Passer par la config hook, comme pour tout hookable natif :

```yaml
sylius_twig_hooks:
    hooks:
        'sylius_admin.order.show.content.header.title_block.actions':
            default_action: # exemple — à adapter selon le hookable natif ciblé
                enabled: false
```

Pour lister les hookables natifs disponibles, consulter :

```bash
grep -rn "sylius_admin.customer.show.content.header" vendor/sylius/sylius/src/Sylius/Bundle/AdminBundle/Resources/config/app/twig_hooks/
```

### 7 — Migration Sylius 1.x → 2.x

Sur 1.x, les action menus des pages de détail passaient par des events Knp Menu type :

- `sylius.menu.admin.order.show`
- `sylius.menu.admin.customer.show`
- `sylius.menu.admin.product.form`

Sur 2.x, ces events sont **dépréciés** (et dans certains cas supprimés). Les remplacements :

| Event 1.x | Remplacement 2.x |
| --- | --- |
| `sylius.menu.admin.order.show` | Twig Hook `sylius_admin.order.show.content.header.title_block.actions` |
| `sylius.menu.admin.customer.show` | Twig Hook `sylius_admin.customer.show.content.header.title_block.actions` |
| `sylius.menu.admin.product.form` | Twig Hooks `sylius_admin.product.{create\|update}.content.form.{side_navigation\|sections}` |
| `sylius.menu.admin.main` | **Inchangé** (Event Listener `MenuBuilderEvent`) |
| `sylius.menu.shop.account` | **Inchangé** (Event Listener `MenuBuilderEvent`) |

Check rapide : si un listener existant écoute un event `sylius.menu.admin.<resource>.<action>` (hors `main`), il est probablement déprécié — migrer vers Twig Hooks.

### 8 — Vérification

```bash
symfony console cache:clear
symfony console debug:event-dispatcher sylius.menu.admin.main    # voir les listeners attachés
symfony console debug:event-dispatcher sylius.menu.shop.account
symfony console debug:twig                                        # erreurs de compilation Twig
```

Puis charger la page visée :

- **Menu sidebar non modifié** → listener non enregistré (YAML oublié), ou event mal orthographié (ex. `sylius.admin.menu.main` au lieu de `sylius.menu.admin.main`).
- **Bouton d'action absent** → cache pas vidé, ou mauvais nom de hookable. Inspecter la page via `CMD+SHIFT+C` pour voir les commentaires `<!-- BEGIN HOOK ... -->` (visibles en `dev`).
- **Onglet produit cliquable mais contenu vide** → divergence `data-bs-target` / `id`. Vérifier que `#product-<nom>` est identique côté bouton et côté pane.
- **Onglet n'apparaît qu'en création** → oublié de déclarer aussi les hookables `update.*.side_navigation` + `update.*.sections`.
- **Icône manquante ou crash `ux_icon() undefined`** → `symfony/ux-icons` pas installé (`composer require symfony/ux-icons`) ou set Tabler manquant (`composer require tabler/icons`).

### 9 — Clôture

Afficher :

- **Fichiers créés/modifiés** :
  - Cas listener : `src/Menu/<Listener>.php`, `config/services.yaml`.
  - Cas hook action : `config/packages/_sylius.yaml` (ou `sylius_twig_hooks.yaml`), `templates/admin/<resource>/show/content/header/title_block/actions/<name>.html.twig`.
  - Cas onglet form : 1 config (4 hookables) + 2 templates (`side_navigation/<name>.html.twig` + `sections/<name>.html.twig`).
- **Commandes de vérif** : `cache:clear`, `debug:event-dispatcher <event>`, `debug:twig`.
- **Ce qui reste** : traductions des labels dans `translations/`, route si le menu pointe vers un controller custom, permission ACL si accès restreint, test fonctionnel sur la page.

## Pièges fréquents

- **Event Listener sans enregistrement YAML** : la classe existe, la méthode est bien écrite, mais rien ne se passe. Le tag `kernel.event_listener` dans `services.yaml` est **obligatoire** pour `MenuBuilderEvent` — pas d'autoconfigure par défaut.
- **Mauvais nom d'event** : `sylius.admin.menu.main` (faux) vs `sylius.menu.admin.main` (bon). L'ordre `menu.<context>.<scope>` est standardisé.
- **Twig Hook pour une sidebar** : ne marchera jamais — la sidebar admin est un Knp Menu, les hooks ne s'y injectent pas. Inverser : Event Listener pour sidebar, Twig Hook pour actions.
- **Onglet produit déclaré seulement sur `create`** : l'onglet apparaît uniquement à la création d'un produit, invisible en édition. Déclarer les 4 hookables (create + update × side_navigation + sections).
- **`setExtra('always_open', true)` sur l'enfant** : sans effet — l'attribut doit être sur le nœud parent (celui avec des enfants).
- **Event 1.x déprécié encore écouté** : `sylius.menu.admin.order.show` sur 2.x → silencieusement ignoré. Migrer vers le Twig Hook équivalent.
- **`data-bs-target` qui pointe nulle part** : typo entre le bouton (`#product-manufacturer`) et le pane (`#product-manufacturers`) — onglet cliquable, contenu vide.
- **Oubli de `cache:clear`** : modification du YAML sans effet, comme pour tout Twig Hook. Réflexe à garder après chaque édition.
- **Icône invisible** : soit `symfony/ux-icons` manquant, soit mauvais namespace (`tabler:star` vs `tabler:stars` — vérifier sur [tabler-icons.io](https://tabler-icons.io/)).
- **Classe CSS oubliée sur le bouton d'action** : sans `dropdown-item`, le bouton apparaît en dehors du dropdown Tabler, mal stylé.

## Argument optionnel

`/sylius:menu admin-main <label> --icon=tabler:<icon> --route=<route>` — ajoute un item au menu principal admin via listener, avec icône Tabler et route.

`/sylius:menu shop-account <label> --icon=tabler:<icon> --route=<route>` — ajoute un item à la sidebar du compte client.

`/sylius:menu action <resource> <name>` — ajoute un bouton d'action via Twig Hook sur la page `show` de la ressource (customer, order, product…).

`/sylius:menu product-tab <name>` — crée un onglet custom sur le form produit (déclare les 4 hookables create/update × side_navigation/sections).

`/sylius:menu product-variant-tab <name>` — idem pour la variante produit.

`/sylius:menu` sans argument — demande le type de menu, l'opération, l'icône, la route, puis guide pas à pas.

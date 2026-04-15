---
name: dynamic
description: Customise un élément dynamique Sylius 2.1+ via Stimulus/Symfony UX : `*_controller.js` dans `assets/<ctx>/controllers/`, binding `stimulus_controller()` Twig, `controllers.json` pour désactiver/remplacer un natif, data-actions, cibles/valeurs.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /dynamic — Customiser les éléments dynamiques Sylius

Tu aides à **ajouter, remplacer ou désactiver un comportement JavaScript** dans l'admin ou le shop Sylius 2.1+. Depuis la 2.1, Sylius adopte **Symfony UX + Stimulus** (Hotwired) : chaque bout d'interactivité est un `*_controller.js` attaché à un élément DOM via `data-controller`. Le chemin officiel : auto-discovery dans `assets/<contexte>/controllers/`, binding via le helper Twig `stimulus_controller()`, injection dans la page via un **Twig Hook** (pas d'override de template vendor).

Référence officielle : [docs.sylius.com/the-customization-guide/customizing-dynamic-elements](https://docs.sylius.com/the-customization-guide/customizing-dynamic-elements). Stimulus : [symfony.com/bundles/StimulusBundle](https://symfony.com/bundles/StimulusBundle/current/index.html). Exemple complet : [github.com/Jibbarth/SyliusCelebratePlugin](https://github.com/Jibbarth/SyliusCelebratePlugin).

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine.
2. Vérifier `sylius/sylius` **≥ 2.1**. En dessous, l'app repose sur l'ancien système d'assets (gulp + Semantic UI côté shop) et ce skill ne s'applique pas. Le PR de migration 2.1 est listé dans la doc : [Sylius-Standard#1126](https://github.com/Sylius/Sylius-Standard/pull/1126).
   - Absent / < 2.1 → *« Ce skill cible Sylius 2.1+ (Symfony UX + Stimulus). Je ne trouve pas la bonne version. On continue quand même ? »*
3. Vérifier la chaîne frontend :
   ```bash
   ls assets/admin/entrypoint.js assets/shop/entrypoint.js assets/controllers.json
   ls assets/admin/controllers.json assets/shop/controllers.json 2>/dev/null
   cat package.json | grep -E '"(@symfony/stimulus-bundle|@hotwired/stimulus|@symfony/ux-|webpack-encore)"'
   ```
   Au minimum : `@hotwired/stimulus` + un `controllers.json` à côté de l'entrypoint du contexte visé.
4. Si la demande est *« changer l'apparence, la couleur, la taille d'un élément »* → ce n'est pas du dynamique, basculer sur **`/sylius:styles`**.
5. Si la demande est *« déplacer, remplacer, injecter un bloc HTML »* → c'est du template, basculer sur **`/sylius:template`** pour le Twig Hook, puis revenir ici pour câbler un controller sur le bloc injecté.
6. Si la demande est *« appel serveur asynchrone avec état réactif »* → regarder **Symfony UX Live Components** (même pattern de binding hookable côté config, annotation `#[AsLiveComponent]` côté PHP), mentionner que la réactivité live dépasse le scope Stimulus pur.

## Règles fondamentales

- **Auto-discovery par défaut, manuel en dernier recours.** Tant que le controller vit sous `assets/<contexte>/controllers/` et s'appelle `*_controller.js`, Symfony UX l'enregistre seul. L'enregistrement manuel (`app.register(...)` dans `bootstrap.js`) n'est justifié que pour : controllers hors dossier standard, import depuis un plugin tiers, ou besoin d'un contrôle explicite sur le moment de load.
- **Le nom du fichier = le nom du controller Stimulus.** `alert_controller.js` → `data-controller="alert"`. `my_widget_controller.js` → `data-controller="my-widget"` (snake → kebab). Tout écart casse le binding silencieusement.
- **Twig Hook pour injecter, pas d'override vendor.** Le controller doit se brancher via un template hookable (`sylius_twig_hooks.hooks.<hook_cible>`). Éditer un template natif pour ajouter un `data-controller` = dette qui saute au prochain update Sylius. Si la zone n'a pas de hook, regarder `/sylius:template` pour en ajouter un ou faire un override ciblé.
- **`controllers.json` pour gouverner les natifs.** Désactiver (`enabled: false`), forcer `lazy` / `eager`, ou remplacer un controller Sylius core se fait *exclusivement* via `assets/<admin|shop>/controllers.json`, pas en monkey-patchant le JS du vendor.
- **Rebuild systématique.** Toute modif JS/JSON côté assets impose `yarn build` (one-shot) ou `yarn watch` (dev continu). Sans ça, `public/build/` reste sur l'ancien bundle — le navigateur ne voit rien.
- **`cache:clear` après rebuild.** Symfony met en cache le manifest d'assets ; sans purge, la page peut pointer sur l'ancien hash. Combo : `yarn build` → `bin/console cache:clear` → hard refresh navigateur (`Cmd+Shift+R`).
- **Pas de logique métier dans le controller.** Stimulus = bindings DOM + événements. Un calcul de prix, une règle promo, un flux de checkout → côté serveur (controller Symfony + Twig Hook), pas dans `connect()`. Stimulus orchestre l'UI, il ne remplace pas le backend.
- **Un controller = une responsabilité.** Plutôt que `mega_product_controller.js` qui gère quantité + wishlist + modal taille, découper en `quantity_controller.js`, `wishlist_controller.js`, `size_modal_controller.js` et les composer sur le même DOM (`data-controller="quantity wishlist"`).

## Déroulement

### 1 — Cadrer le besoin

Demander (ou confirmer) :

- **Contexte** : `admin` (Tabler) ou `shop` (Bootstrap) ?
- **Page cible** : fiche produit, checkout, dashboard admin, grid de commandes, thank-you page, etc.
- **Type d'action** :
  - *Ajouter* un nouveau comportement (quantité +/−, sticky add-to-cart, confirm dialog, shortcut clavier, etc.)
  - *Désactiver* un controller natif Sylius (ex. `taxon-tree`)
  - *Remplacer* un controller natif par le tien (même nom, priorité plus forte)
  - *Charger* un controller existant différemment (`lazy` → `eager` ou l'inverse)
- **Source de données** : statique (hardcodé), issue du DOM (`data-*` attributes), issue du contexte parent Twig (variables passées au template du hookable), ou issue d'un fetch runtime (API Sylius, endpoint custom). Les trois premières sont gratuites côté Stimulus ; la 4e impose un appel HTTP dans le controller (`fetch`, Axios…).

### 2 — Identifier l'élément cible et le hook

Pour brancher un controller sur une zone existante, il faut *d'abord* identifier le hook Twig où injecter le template qui porte le `data-controller`.

**a. Inspecteur navigateur** — sur la page visée, `Cmd+Shift+C` (macOS) / `Ctrl+Shift+C`, survoler la zone. Les commentaires HTML rendus par Twig Hooks encadrent chaque zone :

```html
<!-- BEGIN HOOK | name: "sylius_shop.product.show.content.info.summary" -->
  <!-- BEGIN HOOKABLE | name: "sylius_shop.product.show.content.info.summary.prices" -->
  ...
  <!-- END HOOKABLE -->
<!-- END HOOK -->
```

**b. Symfony Profiler** — onglet `Hooks` pour voir l'arbre complet des hooks rendus, avec les priorités.

**c. Config vendor** (fallback) :

```bash
ls vendor/sylius/sylius/src/Sylius/Bundle/ShopBundle/Resources/config/app/twig_hooks/
ls vendor/sylius/sylius/src/Sylius/Bundle/AdminBundle/Resources/config/app/twig_hooks/
```

Si la zone est déjà dans un hook → parfait, on va y greffer le template du controller. Si elle n'est pas hookable → voir `/sylius:template` §5 pour ouvrir un nouveau hook ou §7 pour override ciblé.

### 3 — Auto-discovery (cas standard)

#### 3.1 — Créer le controller Stimulus

Déposer le fichier sous `assets/<admin|shop>/controllers/` — le nom du fichier détermine le nom du controller.

```javascript
// assets/admin/controllers/alert_controller.js

import { Controller } from '@hotwired/stimulus';

export default class extends Controller {
  connect() {
    alert(this.element.dataset.message || 'Test Alert!');
  }
}
```

Le controller s'enregistre automatiquement sous le nom `alert`.

**Patterns Stimulus courants** à connaître pour structurer proprement :

```javascript
import { Controller } from '@hotwired/stimulus';

export default class extends Controller {
  // Éléments du DOM ciblés
  static targets = ['input', 'output'];

  // Valeurs passées depuis les data-attributes
  static values = {
    min: { type: Number, default: 1 },
    max: Number,
    url: String,
  };

  // Classes CSS togglables
  static classes = ['active', 'loading'];

  connect() {
    // Appelé à chaque attachement au DOM
  }

  disconnect() {
    // Cleanup (timers, listeners globaux, WebSocket)
  }

  increment(event) {
    event.preventDefault();
    const current = parseInt(this.inputTarget.value, 10);
    if (current < this.maxValue) {
      this.inputTarget.value = current + 1;
      this.outputTarget.classList.add(this.activeClass);
    }
  }
}
```

#### 3.2 — Créer le template Twig

Le template porte le `data-controller` via le helper `stimulus_controller()`. Ne jamais écrire `data-controller="alert"` en dur — le helper gère l'ordre et la coexistence de plusieurs controllers.

```twig
{# templates/admin/order/show/alert.html.twig #}

<div {{ stimulus_controller('alert', { message: 'Hello from Sylius!' }) }}></div>
```

Le deuxième argument du helper devient des `data-alert-*-value` — côté JS, ces valeurs sont dispo via `this.messageValue` (si `static values = { message: String }`) ou `this.element.dataset.message` sinon.

Pour combiner avec des targets et des actions :

```twig
<div {{ stimulus_controller('quantity', { min: 1, max: product.stock }) }}>
    <button type="button" data-action="click->quantity#decrement">−</button>
    <input type="number" data-quantity-target="input" value="1">
    <button type="button" data-action="click->quantity#increment">+</button>
    <span data-quantity-target="output"></span>
</div>
```

#### 3.3 — Brancher le template via Twig Hook

```yaml
# config/packages/sylius_twig_hooks.yaml

sylius_twig_hooks:
    hooks:
        'sylius_admin.order.show.content':
            alert:
                template: 'admin/order/show/alert.html.twig'
                priority: 100
```

Pour le shop, exemple d'un compteur de quantité injecté sur la fiche produit entre `prices` (300) et `catalog_promotions` (200) :

```yaml
sylius_twig_hooks:
    hooks:
        'sylius_shop.product.show.content.info.summary':
            quantity_picker:
                template: 'shop/product/show/quantity.html.twig'
                priority: 250
```

#### 3.4 — Rebuild + cache

```bash
yarn build               # one-shot
# ou
yarn watch               # rebuild continu pendant le dev
php bin/console cache:clear
```

Hard refresh navigateur : `Cmd+Shift+R` / `Ctrl+Shift+R`.

### 4 — Désactiver ou gouverner un controller natif

Via `controllers.json` du contexte ciblé. C'est le **seul endroit propre** pour toucher aux controllers Sylius core.

#### 4.1 — Désactiver

Exemple : désactiver le `taxon-tree` de l'admin (cas fréquent pour le remplacer par une version maison).

```json
// assets/admin/controllers.json

{
  "controllers": {
    "@sylius/admin-bundle": {
      "taxon-tree": {
        "enabled": false,
        "fetch": "lazy"
      }
    }
  }
}
```

Rebuild obligatoire. Après ça, même si un `data-controller="taxon-tree"` traîne dans le DOM, plus rien ne se connecte.

#### 4.2 — Forcer eager/lazy

```json
{
  "controllers": {
    "@sylius/shop-bundle": {
      "live-cart": {
        "enabled": true,
        "fetch": "eager"
      }
    }
  }
}
```

- `lazy` (défaut) : le JS du controller n'est téléchargé que quand un `data-controller="..."` matching apparaît dans le DOM. Optimise le poids initial.
- `eager` : téléchargé immédiatement à l'init de la page. Utile si le controller attache des listeners globaux (raccourcis clavier, `popstate`, etc.) sans nécessairement être sur un élément DOM visible dès le premier render.

#### 4.3 — Remplacer par un custom du même nom

Désactiver le natif + déposer un `<nom>_controller.js` dans `assets/<contexte>/controllers/` avec le même nom Stimulus. Le moteur prend le tien.

```json
// assets/admin/controllers.json
{
  "controllers": {
    "@sylius/admin-bundle": {
      "taxon-tree": { "enabled": false }
    }
  }
}
```

```javascript
// assets/admin/controllers/taxon-tree_controller.js
import { Controller } from '@hotwired/stimulus';

export default class extends Controller {
  connect() { /* ma version */ }
}
```

### 5 — Enregistrement manuel (cas avancés)

À réserver aux situations où l'auto-discovery ne suffit pas :

- controller hébergé hors `assets/<contexte>/controllers/` (ex. dans un sous-dossier thématique `assets/shop/checkout/confirm_controller.js`)
- controller fourni par un plugin interne via un package npm privé
- besoin d'exécuter du code custom à l'enregistrement (wiring supplémentaire, transformation du controller)

#### 5.1 — Controller hors dossier standard

```javascript
// assets/shop/custom/confirm_controller.js

import { Controller } from '@hotwired/stimulus';

export default class extends Controller {
  connect() {
    console.log('Confirm controller loaded.');
  }

  onClick(event) {
    event.preventDefault();
    if (!confirm('Confirmer ?')) return;
    event.target.closest('form')?.submit();
  }
}
```

#### 5.2 — Enregistrement dans `bootstrap.js`

```javascript
// assets/shop/bootstrap.js

import ConfirmController from './custom/confirm_controller';

app.register('confirm', ConfirmController);
```

La variable `app` est exposée par `@symfony/stimulus-bundle` : elle pointe sur l'application Stimulus root. Pas besoin de l'importer explicitement dans un `bootstrap.js` fourni par le skeleton Sylius 2.1 — il est pré-câblé.

#### 5.3 — Template + hook

```twig
{# templates/shop/order/confirmation.html.twig #}

<button
    type="submit"
    class="btn btn-primary"
    data-action="click->confirm#onClick"
    {{ stimulus_controller('confirm') }}>
    {{ 'app.order.confirm'|trans }}
</button>
```

```yaml
# config/packages/sylius_twig_hooks.yaml

sylius_twig_hooks:
    hooks:
        'sylius_shop.order.thank_you.content.buttons#customer':
            confirmation:
                template: 'shop/order/confirmation.html.twig'

        'sylius_shop.order.thank_you.content.buttons#guest':
            confirmation:
                template: 'shop/order/confirmation.html.twig'
```

Les deux hooks `#customer` et `#guest` doivent être câblés si le bouton doit s'afficher pour les deux types d'utilisateurs — la thank-you page Sylius a deux variantes.

### 6 — Communiquer avec le serveur

Quand le controller doit déclencher un appel HTTP (ex. update de quantité panier sans reload) :

```javascript
// assets/shop/controllers/cart_item_controller.js
import { Controller } from '@hotwired/stimulus';

export default class extends Controller {
  static values = { url: String };
  static targets = ['input', 'feedback'];

  async update() {
    const response = await fetch(this.urlValue, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json', 'X-Requested-With': 'XMLHttpRequest' },
      body: JSON.stringify({ quantity: parseInt(this.inputTarget.value, 10) }),
    });

    if (!response.ok) {
      this.feedbackTarget.textContent = 'Erreur';
      return;
    }

    const data = await response.json();
    this.feedbackTarget.textContent = `${data.total}`;
  }
}
```

Côté template :

```twig
<div {{ stimulus_controller('cart-item', { url: path('app_cart_item_update', { id: item.id }) }) }}>
    <input type="number" data-cart-item-target="input" data-action="change->cart-item#update">
    <span data-cart-item-target="feedback"></span>
</div>
```

Important : les endpoints doivent renvoyer du JSON côté Symfony controller, et la route doit accepter la method appelée (`PATCH`, `POST`). Pour de la réactivité serveur plus ambitieuse (rerender partiel, validations live), **Symfony UX Live Components** est plus adapté que Stimulus pur — mentionner l'option à l'utilisateur.

### 7 — Coexistence avec autres controllers

Plusieurs controllers sur un même élément :

```twig
<div {{ stimulus_controller('quantity') }} {{ stimulus_controller('analytics', { event: 'qty_change' }) }}>
   ...
</div>
```

Le helper fusionne correctement les `data-controller`. À la main, ça donnerait `data-controller="quantity analytics"` — mais laisser le helper gérer.

Communication inter-controllers : via événements custom DOM (`this.dispatch('changed', { detail: ... })` émis par un controller, `data-action="quantity:changed->analytics#track"` côté DOM) ou via l'application Stimulus (`this.application.getControllerForElementAndIdentifier(...)`). Premier pattern toujours préféré — plus découplé.

### 8 — Vérification

```bash
yarn build                               # pas d'erreur de compilation
php bin/console cache:clear
php bin/console debug:twig-hooks | grep <hook_cible>  # le hookable apparaît
```

Dans le navigateur (DevTools ouvert) :

1. **Onglet Sources / Network** : le bundle `<contexte>.js` contient bien ton controller (chercher le nom, les commentaires `//# sourceMappingURL` aident en dev).
2. **Onglet Elements** : l'élément cible a bien `data-controller="<nom>"`.
3. **Console** : pas d'erreur `Failed to register controller "<nom>"` ni `Controller "<nom>" is not registered`. Un `connect()` avec `console.log` permet de confirmer l'attachement.
4. **Onglet Stimulus** (extension browser Stimulus Devtools si installée) : liste les controllers connectés avec leurs valeurs/targets.

### 9 — Clôture

Afficher :

- **Fichiers créés/modifiés** :
  - `assets/<contexte>/controllers/<nom>_controller.js` (ou un chemin custom + entrée dans `bootstrap.js`)
  - `templates/<contexte>/.../<bloc>.html.twig` (avec `stimulus_controller(...)`)
  - `config/packages/sylius_twig_hooks.yaml` (entrée du hookable)
  - Éventuellement `assets/<contexte>/controllers.json` (pour désactiver/gouverner un natif)
- **Commandes** : `yarn build` (ou `yarn watch`), `php bin/console cache:clear`.
- **Ce qui reste à faire** : traductions des labels affichés par le controller (`translations/messages.<locale>.yaml`), tests E2E (Playwright / Panther) si le comportement est critique, accessibilité (gérer les états `aria-*` si le controller masque/affiche du contenu).

## Pièges fréquents

- **Version Sylius < 2.1** : le `stimulus_controller()`, `controllers.json` et l'auto-discovery n'existent pas. Vérifier `sylius/sylius` ≥ 2.1 avant toute chose — sinon c'est l'ancien pipeline gulp/jQuery, totalement différent.
- **Nom de fichier ≠ nom Stimulus utilisé** : `myAlert_controller.js` → le nom exposé est `my-alert` (camelCase du fichier → kebab du controller), pas `myAlert`. Tester le nom réel avec l'extension Stimulus Devtools si doute.
- **`data-controller` écrit en dur** : `<div data-controller="alert">` au lieu de `{{ stimulus_controller('alert') }}`. Ça marche mais casse la composition dès qu'un second controller arrive sur le même élément, et contourne la gestion des valeurs du helper (qui sérialise proprement les types).
- **Rebuild oublié** : modif JS sans `yarn build` → `public/build/<contexte>.js` garde l'ancien contenu. Symptôme : le controller "n'existe pas", mais le fichier est bien là sur le disque. Lancer `yarn watch` en dev pour éviter le piège.
- **`cache:clear` oublié** : manifest Symfony périmé → navigateur tire l'ancien hash. Rebuild + clear toujours en combo.
- **Hard refresh oublié** : `F5` normal → cache navigateur conserve l'ancien JS/CSS. `Cmd+Shift+R` / `Ctrl+Shift+R` ou DevTools → Network → *Disable cache* en permanence pendant le dev.
- **Controller dans `assets/controllers/` au lieu de `assets/<contexte>/controllers/`** : il sera chargé *dans les deux* contextes (admin et shop), via le fichier `assets/controllers.json` partagé. Généralement non voulu — toujours contextualiser sous `admin/` ou `shop/`.
- **Monkey-patch du vendor pour désactiver un controller natif** : éditer `vendor/sylius/.../controllers.json` → écrasé au prochain `composer update`. Toujours passer par `assets/<contexte>/controllers.json` applicatif, qui override.
- **Mettre de la logique métier dans `connect()`** : calcul de totaux, règles promo, décision checkout. Ça bypass toutes les règles Sylius côté serveur (state machines, processors, promotion rules). Stimulus orchestre l'UI, le backend reste source de vérité.
- **Oublier `disconnect()` pour un controller avec listeners globaux / timers** : `setInterval`, `addEventListener` sur `window`, WebSocket, `IntersectionObserver` non-cleaned → fuite mémoire qui explose après quelques navigations Turbo. Toujours symétriser `connect()` et `disconnect()`.
- **`static values = { url: String }` oublié mais `this.urlValue` utilisé** : Stimulus fait remonter `undefined` sans erreur lisible. Déclarer explicitement chaque value utilisée, avec son type.
- **Dépendance sur un data-attribute non-Stimulus** : `this.element.dataset.foo` lu directement au lieu de `this.fooValue` après déclaration en `static values`. Fonctionne, mais perd la conversion de type auto (Number, Boolean, Array, Object) et la réactivité `fooValueChanged(current, previous)`.
- **Hook branché au mauvais endroit** : template injecté via un hook `sylius_shop.product.show.content` global alors que la cible était `...content.info.summary`. Résultat : l'élément s'affiche, le controller se connecte, mais pas au bon endroit dans le DOM. Toujours vérifier le hook dans le profiler.
- **Oublier la variante `#customer` / `#guest` sur la thank-you page** : le bouton n'apparaît que pour les logged-in ou que pour les guests selon le hook choisi. Les deux hooks existent — utiliser les deux si le comportement doit être universel.

## Argument optionnel

`/sylius:dynamic add admin alert --hook=sylius_admin.order.show.content` — crée un controller `alert` avec son template et son hookable dans l'admin.

`/sylius:dynamic add shop quantity --hook=sylius_shop.product.show.content.info.summary --priority=250` — crée un picker de quantité sur la fiche produit shop, entre `prices` et `catalog_promotions`.

`/sylius:dynamic disable admin taxon-tree` — génère l'entrée `controllers.json` pour désactiver le controller natif `taxon-tree` (package `@sylius/admin-bundle`).

`/sylius:dynamic eager shop live-cart` — force le chargement `eager` d'un controller natif existant.

`/sylius:dynamic replace admin taxon-tree` — désactive le natif + scaffold un `taxon-tree_controller.js` applicatif du même nom.

`/sylius:dynamic manual shop checkout/confirm` — scaffold un controller hors dossier standard + entrée dans `bootstrap.js`.

`/sylius:dynamic` sans argument — demande le contexte (admin/shop), le type d'action (add/disable/eager/replace/manual), le nom du controller et le hook cible, puis guide pas à pas (fichier JS → template Twig → hookable → build → cache).

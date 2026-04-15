---
name: template
description: Customise un template Sylius (shop/admin) via Twig Hooks en priorité (`sylius_twig_hooks`), override `templates/bundles/` en fallback. Couvre hook/hookable, priorités, repérage via Profiler, désactivation, Sylius Themes multi-canal.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /template — Customiser un template Sylius

Tu aides à **modifier ou enrichir un template Sylius** (shop ou admin) sans patcher le vendor. Le pattern officiellement recommandé passe par **Twig Hooks** : des points d'insertion déclarés en YAML dans `config/packages/sylius_twig_hooks.yaml`, qui permettent d'ajouter, réordonner, masquer ou remplacer des blocs sans dupliquer le template d'origine. Deux autres voies existent (override vendor, Sylius Themes) mais sont à réserver à des cas précis.

Référence officielle : [docs.sylius.com/the-customization-guide/customizing-templates](https://docs.sylius.com/the-customization-guide/customizing-templates). Bible hooks : [stack.sylius.com/twig-hooks](https://stack.sylius.com/twig-hooks/getting-started).

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine.
2. Vérifier `sylius/sylius` dans les dépendances.
   - Présent → OK.
   - Absent → *« Ce skill cible Sylius (Twig Hooks + structure `templates/bundles/`). Je ne trouve pas `sylius/sylius`. On continue quand même ? »*
3. Si l'objectif est d'afficher un champ **ajouté via FormType extension** (ex. `secondaryPhoneNumber`), c'est le même pattern : basculer sur **`/sylius:form`** pour la partie form + template, revenir ici uniquement si la question est purement template.
4. Si la demande est *« changer le design de tout le shop pour un canal »* → c'est un cas **Sylius Themes**, pas du hook (cf. §8).

## Règles fondamentales

- **Twig Hooks d'abord, toujours.** L'override de template vendor (`templates/bundles/SyliusShopBundle/...`) duplique le fichier et bloque les updates : toute évolution upstream est perdue, et deux plugins qui overrident le même fichier entrent en conflit silencieusement. Ne l'utiliser **que si aucun hook ne couvre la zone** à modifier.
- **Un hook = un point d'insertion. Un hookable = un bloc de contenu.** Un hook peut contenir autant de hookables qu'on veut, et un hookable peut lui-même contenir d'autres hooks (structure en arbre). Dans l'arbre, seuls les nœuds **feuilles** sont de purs hookables.
- **Hook names = hiérarchiques.** Un `{% hook 'X' %}` rendu depuis un hookable parent se configure sous `parent.X`, pas sous `X`. La concat' est automatique — casser cette hiérarchie explicitement demande `_prefixes: [...]` sur le tag.
- **Priorités par pas de 100.** Les hookables Sylius natifs utilisent `0, 100, 200, 300...`. Un hookable custom sans `priority` tombe à `0` → il sort après les natifs à `0` (ordre de merge). Pour placer un bloc *entre* deux natifs (ex. entre `prices` à `300` et `catalog_promotions` à `200`), utiliser `priority: 250`.
- **Ajouter un bloc ≠ modifier un natif.** Pour retirer un bloc Sylius (ex. `gender` sur le profile update), passer par `enabled: false` — jamais éditer le template Sylius ni supprimer le hookable.
- **Pas de logique lourde dans les templates.** Au-delà de 2-3 lignes de calcul en Twig, sortir sur un **Twig Component** (`bin/console make:twig-component`). Le hookable pointe alors sur `component:` au lieu de `template:`.
- **`cache:clear` systématique** après modif de `sylius_twig_hooks.yaml`. Les hooks sont compilés dans le container : tant que le cache n'est pas vidé, l'ancien rendu reste en place.
- **Pas de chemin vendor en `template:`.** La valeur attend un chemin relatif à `templates/` du projet (ou un alias `@SyliusShop/…`, `@SyliusAdmin/…` si on pointe volontairement vers un template natif pour le rejouer tel quel).

## Déroulement

### 1 — Cadrer le besoin

Demander (ou confirmer) :

- **Contexte** : `shop` (front) ou `admin` (back-office) ?
- **Page cible** : fiche produit, checkout, dashboard admin, profile, grid de commandes, etc.
- **Opération** : ajouter un bloc, réordonner, masquer un bloc natif, remplacer le contenu d'un bloc existant, introduire une nouvelle zone hookable dans un template custom ?
- **Donnée à afficher** : statique, contextuelle (`product`, `order`, `customer`), ou issue d'un service → impacte le choix `template:` vs `component:`.

### 2 — Identifier le hook cible

Deux méthodes fiables (dans l'ordre de rapidité) :

**a. Inspecteur navigateur** — sur la page visée, `CMD+SHIFT+C` (macOS) ou `CTRL+SHIFT+C` (Windows/Linux), survoler la zone proche de l'insertion. Les commentaires HTML rendus par Twig Hooks encadrent chaque zone :

```html
<!-- BEGIN HOOK | name: "sylius_shop.product.show.content.info.summary" -->
  <!-- BEGIN HOOKABLE | name: "sylius_shop.product.show.content.info.summary.prices" -->
  ...
  <!-- END HOOKABLE -->
<!-- END HOOK -->
```

**b. Symfony Profiler** — barre de debug, icône `Hooks` → page dédiée avec l'arbre complet des hooks rendus, priorités, templates pointés. Utile pour tracer une priorité à ajuster ou repérer des hooks imbriqués.

**c. Fichiers de config vendor** (en dernier recours, quand la page n'est pas visitable) :

```bash
# Shop
ls vendor/sylius/sylius/src/Sylius/Bundle/ShopBundle/Resources/config/app/twig_hooks/
# Admin
ls vendor/sylius/sylius/src/Sylius/Bundle/AdminBundle/Resources/config/app/twig_hooks/
```

Grep direct pour un hook donné :

```bash
grep -rn "sylius_shop.product.show.content.info.summary" vendor/sylius/sylius/src/Sylius/Bundle/
```

### 3 — Créer le YAML de config

Fichier canonique : `config/packages/sylius_twig_hooks.yaml` (recommandé). N'importe quel autre `.yaml` sous `config/packages/` fonctionne, mais centraliser évite les hooks dispersés.

Squelette d'ajout d'un bloc *estimated delivery time* sur la fiche produit shop, entre `prices` (priorité `300`) et `catalog_promotions` (priorité `200`) :

```yaml
sylius_twig_hooks:
    hooks:
        'sylius_shop.product.show.content.info.summary':
            estimated_delivery_time:
                template: 'shop/product/show/estimated_delivery_time.html.twig'
                priority: 250
```

Puis créer le template `templates/shop/product/show/estimated_delivery_time.html.twig` :

```twig
<div class="ui label">
    {{ 'app.product.estimated_delivery'|trans }} : 3-5 {{ 'app.time.days'|trans }}
</div>
```

Accès aux données contextuelles : dans un template de hookable, `hookable_metadata.context` expose les variables parent (ex. `hookable_metadata.context.product` sur `sylius_shop.product.show.*`, `hookable_metadata.context.form` sur un form, etc.). `{{ dump(hookable_metadata) }}` est le réflexe pour voir ce qui est dispo.

### 4 — Opérations courantes

#### Ajouter un bloc

Voir §3. Le nom de la hookable est libre (snake_case recommandé, sans conflit avec les natifs listés dans la config vendor).

#### Réordonner sans éditer les templates

```yaml
sylius_twig_hooks:
    hooks:
        'sylius_shop.product.show.content.info.summary':
            average_rating:
                priority: 500 # natif à 400 → remonte au-dessus du header (natif 500 → conflit à arbitrer)
            prices:
                priority: 450
```

Seule la clé `priority:` est nécessaire — pas besoin de redéclarer `template:` ou `component:` du natif. Sylius merge les configs et ne modifie que la priorité.

#### Masquer un bloc natif

```yaml
sylius_twig_hooks:
    hooks:
        'sylius_shop.account.profile_update.update.content.main.form.additional_information':
            gender:
                enabled: false
```

`enabled: false` coupe le rendu côté hook. Le champ reste soumis si le form le contient — pour le retirer aussi côté form, passer par **`/sylius:form`**.

#### Remplacer le template d'un hookable natif

```yaml
sylius_twig_hooks:
    hooks:
        'sylius_shop.product.show.content.info.summary':
            prices:
                template: 'shop/product/show/custom_prices.html.twig'
                # priority: conservée (300 natif), pas besoin de redéclarer
```

Le nouveau template doit **respecter le contrat** du hookable natif (mêmes données attendues par les hooks descendants), sinon l'arborescence casse en dessous. Si les descendants doivent aussi être rejoués, ton template inclut explicitement les `{% hook ... %}` nécessaires.

### 5 — Introduire un nouveau hook dans un template custom

Dans un template applicatif (ex. landing page custom), déclarer un point d'extension ouvert aux futurs hookables :

```twig
{# templates/shop/homepage/index.html.twig #}
<section class="hero">
    {% hook 'hero' %}
</section>
```

Par défaut, ce hook est nommé selon la hiérarchie parente (vide ici → juste `hero`). La config :

```yaml
sylius_twig_hooks:
    hooks:
        'hero':
            slogan:
                template: 'shop/homepage/hero/slogan.html.twig'
                priority: 200
            cta:
                template: 'shop/homepage/hero/cta.html.twig'
                priority: 100
```

Cas hiérarchique (le hook est rendu depuis un hookable parent existant) :

```yaml
sylius_twig_hooks:
    hooks:
        'app.homepage.content':
            hero_section:
                template: 'shop/homepage/hero.html.twig' # ce template contient {% hook 'hero' %}
                priority: 100
        'app.homepage.content.hero_section.hero': # nom complet
            slogan:
                template: 'shop/homepage/hero/slogan.html.twig'
```

#### Forcer un nom explicite avec `_prefixes`

Si tu veux casser l'héritage (ex. mutualiser un hook depuis plusieurs contextes parents) :

```twig
<div id="container">
    {% hook 'hero' with { _prefixes: ['app_landing'] } %}
</div>
```

Config :

```yaml
sylius_twig_hooks:
    hooks:
        'app_landing.hero':
            slogan:
                template: 'shop/landing/hero/slogan.html.twig'
```

À réserver aux cas où la mutualisation vaut la surcharge cognitive — sinon la hiérarchie implicite est plus lisible.

### 6 — Hookable en Twig Component

Dès qu'un hookable demande de la logique (appel service, requête DB, calcul conditionnel), sortir sur un Twig Component plutôt que bourrer le `.html.twig` :

```bash
bin/console make:twig-component EstimatedDelivery
```

Génère :

```php
// src/Twig/Components/EstimatedDelivery.php
namespace App\Twig\Components;

use Symfony\UX\TwigComponent\Attribute\AsTwigComponent;
use Sylius\Component\Core\Model\ProductInterface;

#[AsTwigComponent]
final class EstimatedDelivery
{
    public ProductInterface $product;

    public function getDeliveryRange(): string
    {
        return $this->product->isInStock() ? '3-5 days' : '10-15 days';
    }
}
```

```twig
{# templates/components/EstimatedDelivery.html.twig #}
<div class="ui label">
    {{ 'app.product.estimated_delivery'|trans }} : {{ this.deliveryRange }}
</div>
```

Config hookable :

```yaml
sylius_twig_hooks:
    hooks:
        'sylius_shop.product.show.content.info.summary':
            estimated_delivery_time:
                component: 'EstimatedDelivery'
                props:
                    product: '@=_context.product'
                priority: 250
```

- `props:` injecte les propriétés typées du component. L'expression `@=_context.product` passe la variable `product` du contexte parent.
- Si le nom du component n'est pas évident : `bin/console debug:twig-component`.
- Un component **Live** (UX Live Component) suit le même pattern côté hookable — la réactivité est gérée par l'annotation `#[AsLiveComponent]` côté PHP.

### 7 — Override de template vendor (dernier recours)

Utilisable **seulement** quand aucun hook ne couvre la zone. Le pattern Symfony standard est `templates/bundles/<BundleName>/<chemin>.html.twig`.

Exemple : custom de la page login shop.

- **Template natif** : `vendor/sylius/sylius/src/Sylius/Bundle/ShopBundle/Resources/views/login.html.twig` (alias `@SyliusShop/login.html.twig`).
- **Chemin d'override** : `templates/bundles/SyliusShopBundle/login.html.twig`.

```twig
{% extends '@SyliusShop/layout.html.twig' %}
{% import '@SyliusUi/Macro/messages.html.twig' as messages %}

{% block content %}
<div class="ui column stackable center page grid">
    {% if last_error %}
        {{ messages.error(last_error.messageKey|trans(last_error.messageData, 'security')) }}
    {% endif %}
    <h1>Mon login custom</h1>
    <form class="ui form" action="{{ path('sylius_shop_login_check') }}" method="post" novalidate>
        {{ form_row(form._username, {'value': last_username|default('')}) }}
        {{ form_row(form._password) }}
        <button type="submit" class="ui fluid primary button">{{ 'sylius.ui.login_button'|trans }}</button>
    </form>
</div>
{% endblock %}
```

Points clés :

- Le dossier `SyliusShopBundle` (sans le préfixe `@`) est le nom complet du bundle tel que déclaré par Sylius. Pour l'admin : `SyliusAdminBundle`.
- Tant que possible, **continuer d'étendre** le layout natif via `{% extends '@SyliusShop/...' %}` et redéfinir uniquement les `{% block %}` pertinents — ça limite la dérive lors des updates.
- Dupliquer entièrement un template = dette. Ajouter un commentaire en tête du fichier rappelant la version Sylius d'origine copiée, pour faciliter les future merges.
- `cache:clear` obligatoire après création d'un override.

### 8 — Sylius Themes (multi-canal)

Pour des designs différenciés par canal (canal B2C vs canal B2B, ou multi-marques sur une instance), Sylius Themes permet de rattacher un répertoire `themes/<slug>/` à un canal.

Déroulement rapide :

1. `composer require sylius/theme-bundle` (déjà installé dans Sylius standard).
2. Créer `themes/<slug>/composer.json` + l'arborescence `themes/<slug>/SyliusShopBundle/views/...`.
3. Rattacher le thème au canal dans l'admin Sylius (ou en fixture).
4. Les templates de `themes/<slug>/` remplacent ceux du vendor **pour ce canal uniquement**.

Doc officielle Themes : [docs.sylius.com/the-book/frontend-and-themes](https://docs.sylius.com/the-book/frontend-and-themes). Bundle : [github.com/Sylius/SyliusThemeBundle](https://github.com/Sylius/SyliusThemeBundle/blob/master/docs/index.md).

C'est un mécanisme **orthogonal** aux Twig Hooks : un thème peut lui-même utiliser ses propres hooks/hookables, et vice versa. Pour une simple surcharge sur un canal unique, les Twig Hooks suffisent presque toujours.

### 9 — Variables globales Twig

Toutes les vues Sylius reçoivent la variable `sylius` (depuis `ShopperContext`) :

| Variable Twig         | Méthode ShopperContext |
| --------------------- | ---------------------- |
| `sylius.channel`      | `getChannel()`         |
| `sylius.currencyCode` | `getCurrencyCode()`    |
| `sylius.localeCode`   | `getLocaleCode()`      |
| `sylius.customer`     | `getCustomer()`        |

Usage fréquent dans un hookable : router le contenu selon le canal, la locale ou l'état de connexion du client :

```twig
{% if sylius.channel.code == 'FR_WEB' %}
    <p>{{ 'app.channel.fr_web.promo_banner'|trans }}</p>
{% endif %}
```

Débogage : `{{ dump(sylius.channel) }}` pour inspecter le canal en cours.

### 10 — Vérification

```bash
symfony console cache:clear
symfony console debug:twig                # vérifier qu'aucune erreur de compilation
```

Puis charger la page dans le navigateur :

- **Bloc absent** → `cache:clear` oublié, ou mauvais nom de hook (hiérarchie parent non prise en compte).
- **Bloc au mauvais endroit** → priorité à recalibrer. Vérifier dans le profiler l'ordre réel.
- **Commentaires HTML `BEGIN HOOK` manquants** → les hooks ne s'impriment pas si l'environnement est en `prod`. Debug en `dev`.
- **`component: 'Xxx'` ne rend rien** → `bin/console debug:twig-component` pour vérifier l'ID, et que la classe est bien découverte (pas de typo `#[AsTwigComponent]`).
- **Override vendor qui ne prend pas** → vérifier le chemin exact `templates/bundles/<BundleName>/...` et l'absence de faute sur le nom du bundle (ex. `SyliusShopBundle` et non `SyliusShop`).

### 11 — Clôture

Afficher :

- **Fichiers créés/modifiés** : `config/packages/sylius_twig_hooks.yaml`, `templates/<chemin>/<bloc>.html.twig`, éventuellement `src/Twig/Components/<Component>.php` + son template, ou `templates/bundles/<Bundle>/<chemin>.html.twig` pour un override.
- **Commandes de vérif** : `cache:clear`, `debug:twig`, éventuellement `debug:twig-component`.
- **Ce qui reste à faire** : traductions des labels dans `translations/`, CSS/Sass si le bloc a un style custom, tests fonctionnels sur la page.

## Pièges fréquents

- **Override vendor pour un simple ajout** : réflexe hérité de Sylius 1.x. Sur 2.x, 95 % des customs passent par Twig Hooks — vérifier qu'aucun hook ne couvre la zone avant de copier un template.
- **Priorité oubliée** : hookable ajouté sans `priority:` → tombe à `0`, s'affiche *après* le natif à `0` (`short_description` par exemple). Toujours regarder le natif environnant et placer entre deux multiples de 100.
- **Hook hiérarchique raté** : configurer `'hook_name'` alors que le hook est rendu depuis un hookable parent → la config ne matche jamais. Recoller la hiérarchie complète : `'parent_hookable_name.hook_name'`.
- **Modifier un template dans `vendor/sylius/`** : patch volatile, saute au prochain `composer update`. Toujours `templates/bundles/...` ou hook.
- **Cache pas vidé** : les hooks sont compilés dans le container. Tant que `cache:clear` n'est pas exécuté, les modifs de YAML n'ont aucun effet visible.
- **`enabled: false` sur un hook au lieu d'un hookable** : coupe toute la zone et tout son arbre descendant. Cibler le hookable précis, pas son parent.
- **Component avec props non typées** : si la classe PHP déclare `public ProductInterface $product;` mais le YAML oublie `props.product`, Symfony UX lève une erreur d'hydratation au rendu. Toujours aligner YAML `props:` et propriétés PHP.
- **`_prefixes` utilisé en premier réflexe** : la plupart du temps, la hiérarchie implicite suffit. `_prefixes` est là pour les cas exotiques de mutualisation — s'en servir par défaut rend la config cryptique.
- **Double accolade de dump oublié en prod** : `{{ dump(hookable_metadata) }}` laissé en place → erreur en `prod` (Twig `dump` absent sauf `twig/twig:^3` avec bundle debug). Retirer avant commit.

## Argument optionnel

`/sylius:template add sylius_shop.product.show.content.info.summary estimated_delivery_time --priority=250` — cadre l'ajout d'un hookable entre `prices` et `catalog_promotions` sur la fiche produit shop.

`/sylius:template remove sylius_shop.account.profile_update.update.content.main.form.additional_information gender` — désactive le hookable `gender` via `enabled: false`.

`/sylius:template reorder sylius_shop.product.show.content.info.summary` — propose une table des priorités actuelles et aide à réordonner.

`/sylius:template override SyliusShopBundle login.html.twig` — bascule explicitement sur le pattern d'override vendor pour la page de login.

`/sylius:template` sans argument — demande le contexte (shop/admin), la page cible, l'opération (add/remove/reorder/override/new-hook), puis guide pas à pas.

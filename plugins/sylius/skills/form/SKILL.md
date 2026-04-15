---
name: form
description: Étend un FormType Sylius via `AbstractTypeExtension` : ajoute/retire un champ sur `sylius_shop.form.type.*` ou `sylius_admin.form.type.*`, priorités, champs dynamiques (`PRE_SET_DATA`), Twig Hook pour l'affichage. Champ absent → `/sylius:model`.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /form — Customiser un formulaire Sylius

Tu aides à **étendre un `FormType` Sylius natif** (ajouter, modifier, retirer des champs) sans toucher au vendor. Le pattern officiel passe par une classe `AbstractTypeExtension` taguée `form.type_extension`, couplée à une mise à jour du template via **Twig Hooks** pour rendre réellement le nouveau champ dans le HTML.

Référence officielle : [docs.sylius.com/the-customization-guide/customizing-forms](https://docs.sylius.com/the-customization-guide/customizing-forms).

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine.
2. Vérifier `sylius/sylius` dans les dépendances.
   - Présent → OK.
   - Absent → *« Ce skill cible Sylius (extension de FormType via `form.type_extension`). Je ne trouve pas `sylius/sylius`. On continue quand même ? »*
3. Si le champ ajouté n'existe pas encore sur l'entité (ex. `secondaryPhoneNumber` sur `Customer`) → basculer d'abord sur **`/sylius:model`** pour l'ajouter au modèle + générer la migration, puis revenir ici pour l'exposer dans le form.

## Règles fondamentales

- **Ne jamais étendre la forme `sylius.form.type.*`** sauf besoin explicite de toucher **les deux contextes** shop et admin en même temps. C'est la **base** des deux autres : toute extension s'applique partout. Dans 95 % des cas, on veut `sylius_shop.form.type.*` **ou** `sylius_admin.form.type.*`, pas les deux.
- **Un form extension = une cible précise** : `getExtendedTypes()` retourne la classe réelle résolue par `debug:container` (ex. `Sylius\Bundle\ShopBundle\Form\Type\CustomerProfileType`), pas l'interface ni un alias de service.
- **Ajouter un champ au form ne l'affiche pas** : Sylius utilise Twig Hooks, donc même si `buildForm()` ajoute le champ, il faut créer un template hookable et l'enregistrer dans `config/packages/twig_hooks.yaml` pour qu'il apparaisse réellement. Un champ "invisible malgré tout" = hook oublié.
- **Autoconfigure = par défaut** sur Sylius 2.x / Symfony 6+ : pas besoin de tag manuel dans `services.yaml`. Tag `form.type_extension` requis seulement si `_defaults: autoconfigure: true` est désactivé, ou si tu fixes explicitement une **priorité**.
- **Forms déjà étendus dans le Core** (ex. `ProductVariantType` étendu par `ProductVariantTypeExtension` dans `CoreBundle`) : déclarer ta propre extension avec une priorité différente — les priorités **plus élevées** passent en premier.
- **Champs dynamiques** (ajoutés par event listener `PRE_SET_DATA`) : ne s'enlèvent pas avec `->remove()` dans `buildForm()` — il faut un event listener sur le **même événement**, sinon le champ est remis après coup.
- **Labels custom → traductions** : chaque label custom (`app.form.customer.surname`) doit exister dans `translations/messages.<locale>.yaml`, sinon l'UI affiche la clé brute.

## Déroulement

### 1 — Cadrer le besoin

Demander (ou confirmer si déjà fourni) :

- **Contexte cible** : `shop` (front client), `admin` (back-office), ou les deux (rare).
- **Form concerné** : `CustomerProfileType`, `AddressType`, `ProductType`, `ProductVariantType`, `CheckoutAddressType`, etc.
- **Opérations** : ajout de champ(s), modif de label/options, suppression de champ.
- **Champ existant sur l'entité ?** Si non, rediriger vers `/sylius:model` pour l'ajouter avant.

### 2 — Repérer la classe à étendre

```bash
symfony console debug:container | grep form.type.<snake_case_du_form>
```

Exemple pour `CustomerProfileType` :

```bash
symfony console debug:container | grep form.type.customer_profile
```

Retour typique :

```
sylius.form.type.customer_profile         → Sylius\Bundle\CustomerBundle\Form\Type\CustomerProfileType
sylius_shop.form.type.customer_profile    → Sylius\Bundle\ShopBundle\Form\Type\CustomerProfileType
sylius_admin.form.type.customer_profile   → Sylius\Bundle\AdminBundle\Form\Type\CustomerProfileType (si présent)
```

Règle de lecture :

| Préfixe service        | À utiliser quand…                                        |
| ---------------------- | -------------------------------------------------------- |
| `sylius.form.type.*`   | tu veux impacter **shop + admin** en même temps (rare)    |
| `sylius_shop.form.*`   | customisation **shop uniquement** — cas le plus fréquent |
| `sylius_admin.form.*`  | customisation **admin uniquement**                       |

### 3 — Créer la form extension

Exemple : ajouter `secondaryPhoneNumber`, retirer `gender`, renommer le label de `lastName` sur le profil client shop — `src/Form/Extension/CustomerProfileTypeExtension.php` :

```php
<?php

declare(strict_types=1);

namespace App\Form\Extension;

use Sylius\Bundle\ShopBundle\Form\Type\CustomerProfileType;
use Symfony\Component\Form\AbstractTypeExtension;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;

final class CustomerProfileTypeExtension extends AbstractTypeExtension
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add('secondaryPhoneNumber', TextType::class, [
                'required' => false,
                'label' => 'app.form.customer.secondary_phone_number',
            ])
            ->remove('gender')
            ->add('lastName', TextType::class, [
                'label' => 'app.form.customer.surname',
            ]);
    }

    public static function getExtendedTypes(): iterable
    {
        return [CustomerProfileType::class];
    }
}
```

Points de vigilance :

- **`final class`** : pas d'héritage de l'extension ailleurs, évite des surprises de résolution.
- **`declare(strict_types=1)`** + signatures typées (`void`, `iterable`).
- **`add()` d'un champ existant** = **modification** des options (label, contraintes, CSS), pas un ajout en double — Symfony fusionne.
- **`remove()`** fonctionne seulement pour les champs ajoutés dans `buildForm()`. Pour un champ ajouté dynamiquement (event listener), voir §7.
- **`getExtendedTypes()`** doit retourner un `iterable` (tableau suffit), avec la/les classe(s) FQCN.

### 4 — Enregistrer l'extension

**Cas autoconfigure (défaut Sylius 2.x)** — rien à faire, Symfony découvre la classe automatiquement via `AbstractTypeExtension`.

**Cas autoconfigure désactivé** — `config/services.yaml` :

```yaml
services:
    app.form.extension.type.customer_profile:
        class: App\Form\Extension\CustomerProfileTypeExtension
        tags:
            - { name: form.type_extension }
```

Vérifier l'enregistrement :

```bash
symfony console debug:form "Sylius\Bundle\ShopBundle\Form\Type\CustomerProfileType"
```

La commande doit lister ton extension dans la section `Type Extensions`. Si elle n'apparaît pas :

- mauvais `getExtendedTypes()` (classe Sylius non-résolue ou erreur de namespace) ;
- cache Symfony non vidé après création de la classe (`symfony console cache:clear`) ;
- `autoconfigure: false` sans tag manuel.

### 5 — Mettre à jour le template via Twig Hooks

Ajouter un champ dans le `FormType` **ne suffit pas** : Sylius rend les forms via des Twig Hooks, donc il faut pousser un template pour le nouveau champ et désactiver le hook du champ supprimé.

#### Identifier le hook parent

Ouvrir la page qui contient le form dans un navigateur, `Inspecter`, et chercher les commentaires HTML :

```html
<!-- BEGIN HOOK | name: "sylius_shop.account.profile_update.update.content.main.form" -->
```

Le nom du hook est ce qu'il faut viser.

#### Créer le template du nouveau champ

`templates/account/profile_update/update/content/main/form/secondary_phone_number.html.twig` :

```twig
<div>{{ form_row(hookable_metadata.context.form.secondaryPhoneNumber) }}</div>
```

#### Enregistrer le template sur le hook

`config/packages/twig_hooks.yaml` :

```yaml
sylius_twig_hooks:
    hooks:
        'sylius_shop.account.profile_update.update.content.main.form':
            secondary_phone_number:
                template: 'account/profile_update/update/content/main/form/secondary_phone_number.html.twig'
                priority: 600
```

La `priority` contrôle l'ordre d'affichage dans le form (plus haut = plus tôt dans le rendu). Valeurs usuelles dans Sylius : `100` à `700` par pas de `100`.

#### Désactiver le hook du champ supprimé

Pour `gender`, trouver le hook qui le rend (souvent dans `…additional_information`) et le désactiver :

```yaml
sylius_twig_hooks:
    hooks:
        'sylius_shop.account.profile_update.update.content.main.form.additional_information':
            gender:
                enabled: false
```

Docs hooks : [stack.sylius.com/twig-hooks/getting-started](https://stack.sylius.com/twig-hooks/getting-started).

### 6 — Cas d'un form déjà étendu dans le Core

Certains forms sont **déjà** étendus par Sylius lui-même (ex. `ProductVariantType` étendu par `ProductVariantTypeExtension` dans `Sylius\Bundle\CoreBundle\Form\Extension\`). Pour ajouter **ta propre** extension par-dessus, définir une priorité :

```yaml
services:
    app.form.extension.type.product_variant:
        class: App\Form\Extension\ProductVariantTypeMyExtension
        tags:
            - { name: form.type_extension, extended_type: Sylius\Bundle\ProductBundle\Form\Type\ProductVariantType, priority: -5 }
```

- **Priorité plus élevée = exécute en premier**. Une priorité négative (comme `-5`) passe **après** l'extension Core (priorité par défaut `0`), ce qui est en général souhaitable — tu modifies un état déjà rempli par Sylius.
- `extended_type` dans le tag est **obligatoire** si `getExtendedTypes()` renvoie plusieurs types ou si tu veux overrider le type cible du tag.

### 7 — Champs ajoutés dynamiquement (event listeners)

Certains champs ne sont pas déclarés dans `buildForm()` mais ajoutés via `FormEvents::PRE_SET_DATA` (ex. `channelPricings` sur `ProductVariantType`). Ils ne se retirent **pas** avec un simple `->remove()` statique — Symfony les rajoute après ton extension.

#### Pattern de retrait

```php
use Symfony\Component\Form\FormEvent;
use Symfony\Component\Form\FormEvents;

public function buildForm(FormBuilderInterface $builder, array $options): void
{
    $builder->addEventListener(FormEvents::PRE_SET_DATA, function (FormEvent $event): void {
        $event->getForm()->remove('channelPricings');
    });
}
```

#### Pattern d'ajout dynamique

Même principe pour ajouter un champ qui dépend de la donnée hydratée (ex. un champ qui dépend du `ProductVariant` courant) :

```php
$builder->addEventListener(FormEvents::PRE_SET_DATA, function (FormEvent $event): void {
    $variant = $event->getData();
    // ... ajouter un champ en fonction de $variant
});
```

Le listener **remplace** le comportement dynamique de Sylius seulement si ta priorité d'extension est supérieure à celle de l'extension Core (cf. §6).

### 8 — Vérification finale

```bash
symfony console cache:clear
symfony console debug:form "Sylius\Bundle\ShopBundle\Form\Type\CustomerProfileType"
```

Puis ouvrir la page concernée dans le navigateur :

- Le nouveau champ s'affiche ? → form extension + Twig Hook OK.
- Il est dans le DOM mais invisible ? → CSS ou ordre du Twig Hook (priorité).
- Il apparaît en doublon ? → un autre hook le rend déjà quelque part, inspecter `debug:twig-hooks` (Sylius ≥ 2.0).
- Il est soumis mais pas persisté ? → `mapped: false` à retirer, ou champ absent de l'entité (→ `/sylius:model`).

### 9 — Clôture

Afficher :

- **Fichiers créés/modifiés** : `src/Form/Extension/<Form>TypeExtension.php`, `templates/<chemin>/<champ>.html.twig`, `config/packages/twig_hooks.yaml`, éventuellement `config/services.yaml` et `translations/messages.*.yaml`.
- **Commandes de vérif** : `debug:container`, `debug:form`, `cache:clear`.
- **Ce qui reste à faire** : ajouter les traductions des labels, tests fonctionnels (submit du form), grid admin si le champ doit apparaître en listing.

## Pièges fréquents

- **Étendre `sylius.form.type.*` par erreur** : la modif s'applique aux deux contextes (shop **et** admin), souvent pas ce qu'on veut. Si tu hésites, c'est presque toujours `sylius_shop.form.type.*` ou `sylius_admin.form.type.*` qu'il faut viser.
- **Champ ajouté au form mais invisible en page** : Twig Hook non déclaré ou priorité absente. Le `buildForm()` fait exister la donnée côté Symfony, mais c'est le hook qui la rend côté HTML.
- **`remove()` inefficace** : le champ est dynamique (event listener Core). Passer par un `addEventListener(PRE_SET_DATA, …)` pour le retirer au bon moment du cycle.
- **Extension Core écrasée** : tu override un form déjà étendu sans priorité → ton extension passe avant celle de Sylius et les champs dynamiques Core disparaissent. Fixer une priorité négative (`-5` à `-10`) rétablit l'ordre.
- **Label custom brut dans l'UI** : `app.form.customer.surname` non déclaré dans `translations/`. Ajouter l'entrée dans le fichier de la locale active.
- **`getExtendedTypes()` renvoyant une interface** : ne matche aucun type concret — l'extension n'est jamais appliquée. Toujours retourner le FQCN concret résolu par `debug:container`.
- **Cache pas vidé après création** : l'extension existe sur disque mais Symfony continue d'utiliser le container compilé. `symfony console cache:clear` systématique après création d'un service autoconfiguré.
- **Modifier le template natif** dans `vendor/sylius/` : patch volatile qui saute au prochain `composer update`. Passer exclusivement par Twig Hooks, jamais par override de template vendor.

## Argument optionnel

`/sylius:form CustomerProfileType secondaryPhoneNumber:text` — étend le profil client shop avec un champ texte optionnel.

`/sylius:form AddressType company:text?` — ajoute un champ `company` nullable sur `AddressType`.

`/sylius:form ProductVariantType --priority=-5` — prépare une extension avec priorité négative pour un form déjà étendu par le Core.

`/sylius:form` sans argument — demande le contexte (shop/admin), le form cible et les opérations (add/remove/modify) puis guide pas à pas.

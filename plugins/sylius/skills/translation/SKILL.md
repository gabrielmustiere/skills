---
name: translation
description: Customise les libellés/messages traduits Sylius via `translations/messages.<locale>.yaml` (ou `validators`, `flashes`, `security`), repérage via Profiler Symfony, priorité par domaine/locale. Message de contrainte → `/sylius:validation`.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /translation — Customiser les traductions Sylius

Tu aides à **redéfinir un libellé traduit** (label de form, texte de bouton, message flash, message de validation) dans un projet Sylius **sans patcher le vendor**. Le pattern officiel : créer / éditer `translations/<domaine>.<locale>.yaml` à la racine du projet (ou du plugin) en reprenant **la même clé** que celle déclarée par le bundle Sylius concerné. Symfony merge les catalogues et l'override applicatif gagne.

Référence officielle : [docs.sylius.com/the-customization-guide/customizing-translations](https://docs.sylius.com/the-customization-guide/customizing-translations).

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine.
2. Vérifier `sylius/sylius` dans les dépendances.
   - Présent → OK.
   - Absent → *« Ce skill cible Sylius (override du catalogue `translations/` appliqué par le Translator Symfony). Je ne trouve pas `sylius/sylius`. On continue quand même ou on bascule sur `/symfony:translation` pour un projet Symfony générique ? »*
3. Si la demande est *« ajouter des champs multilingues à une entité »* (pas surcharger un libellé UI) → basculer sur **`/sylius:translation-entity`** : c'est un pattern Doctrine (`TranslatableTrait`, `*Translation`), rien à voir avec le catalogue de messages.
4. Si la demande concerne un **message d'erreur de contrainte** Symfony (`#[Assert\Length]`, etc.) → la clé vit en général sous le domaine `validators` et la contrainte elle-même se redéfinit via **`/sylius:validation`**. Revenir ici seulement pour traduire la clé dans les locales.

## Règles fondamentales

- **Un fichier par locale, par domaine** : la convention Symfony est `<domaine>.<locale>.<format>`. Sylius utilise principalement quatre domaines :
  - `messages` → libellés UI, titres, boutons, labels de form (domaine par défaut).
  - `validators` → messages d'erreur de contraintes Symfony / Sylius.
  - `flashes` → messages flash après action (`sylius.product.create`, etc.).
  - `security` → messages liés à l'authentification (Symfony Security).
  Ne pas mélanger : un libellé de bouton sous `validators.en.yaml` ne sera jamais chargé par le form qui regarde `messages`.
- **L'override reprend exactement la même clé** que le vendor. Le Translator Symfony merge les catalogues avec un ordre de priorité **application > plugins > bundles vendor** : la dernière source qui déclare la clé gagne. Pas besoin de redéclarer les clés voisines — seule la clé override est écrite dans le fichier applicatif, le reste continue de venir du vendor.
- **Les clés Sylius sont structurées en YAML imbriqué**, mais référencées à plat dans le code (`sylius.form.customer.email`). Respecter l'arborescence exacte du vendor, sinon la clé n'est pas matchée :
  ```yaml
  sylius:
      form:
          customer:
              email: Username   # override de sylius.form.customer.email
  ```
- **Une clé par locale à maintenir** : si le projet supporte `en`, `fr`, `pl`, et que tu override `sylius.form.customer.email`, il faut écrire l'override dans **chaque** `messages.<locale>.yaml` que tu veux voir affecté. Sinon seuls les visiteurs de la locale modifiée voient le libellé custom ; les autres retombent sur la traduction vendor (ou la `fallback_locale` si la locale n'est pas couverte du tout).
- **Fallback locale** (`translator.fallbacks` dans `config/packages/translation.yaml`, ou `framework.default_locale`) s'applique **clé par clé** : si `messages.pl.yaml` n'a pas la clé et que le fallback est `en`, Symfony remonte la `en`. Utile en dev, piège en prod : une clé oubliée en polonais s'affichera en anglais sans warning.
- **Cache obligatoire** après toute modif des fichiers `translations/` en `prod` : `php bin/console cache:clear`. En `dev`, le Translator recharge automatiquement (watcher sur le dossier) — mais si rien ne change à l'écran, vider quand même.
- **Override dans un plugin** : un plugin Sylius qui embarque des traductions les place dans `src/<Plugin>/Resources/translations/<domaine>.<locale>.yaml` (ou `translations/` à la racine du bundle selon la version). Depuis l'application, on peut **re-override** les clés du plugin dans `translations/` de l'app — l'ordre de priorité reste `app > plugin > core`.
- **Clés introuvables dans le YAML vendor** : certaines chaînes viennent du code PHP (exceptions, messages flash construits à la volée, attributs `#[Groups]`). Elles sont toujours repérables via le Profiler ; ne pas les chercher dans les sources YAML du bundle.

## Déroulement

### 1 — Cadrer le besoin

Demander (ou confirmer si déjà fourni) :

- **Chaîne source** à remplacer (ex. `"Email"`, `"Add to cart"`, `"Last name"`).
- **Nouvelle chaîne** à afficher (ex. `"Username"`, `"Buy now"`, `"Surname"`).
- **Locale(s) concernée(s)** : `en`, `fr`, `pl`, etc. Lister toutes les locales actives du projet (voir `config/packages/translation.yaml` → `framework.enabled_locales` ou `framework.translator.fallbacks`).
- **Portée** : shop, admin, ou les deux ? (Certaines clés sont partagées — `sylius.form.address.street` vit des deux côtés — d'autres sont scoped sur un bundle.)
- **Domaine probable** : label / bouton → `messages`. Erreur de validation → `validators`. Message flash post-action → `flashes`. Message de login → `security`.

### 2 — Repérer la clé exacte via Symfony Profiler

Le chemin rapide, imposé par la doc Sylius :

1. Lancer le serveur de dev : `symfony serve -d` (ou `symfony server:start`).
2. Ouvrir la page qui affiche le libellé à changer.
3. Cliquer sur la **toolbar Symfony** en bas → onglet **Translations** (icône globe).
4. Filtrer la colonne « Message » sur le texte à modifier → lire la **clé** et le **domaine** exacts dans les colonnes de gauche.

Alternative ligne de commande, utile pour valider qu'une clé est bien déclarée :

```bash
# Chercher une clé dans tous les catalogues chargés pour une locale
php bin/console debug:translation en --domain=messages | grep -i "email"

# Lister les clés manquantes dans une locale (par rapport au fallback)
php bin/console debug:translation pl --only-missing
```

Alternative repository : grep direct dans les YAML vendor du bundle concerné.

```bash
# Pour une clé shop/checkout/customer form
grep -r "Email" vendor/sylius/sylius/src/Sylius/Bundle/CustomerBundle/Resources/translations/
```

### 3 — Créer (ou éditer) le fichier `translations/<domaine>.<locale>.yaml`

Exemple : remplacer `"Email"` par `"Username"` sur le form client, en anglais.

```yaml
# translations/messages.en.yaml

sylius:
    form:
        customer:
            email: Username
```

Puis, pour couvrir le français :

```yaml
# translations/messages.fr.yaml

sylius:
    form:
        customer:
            email: Nom d'utilisateur
```

Points de vigilance :

- **Respecter l'indentation YAML** à 4 espaces comme les fichiers Sylius — un désalignement d'un niveau casse le merge (la clé ne matche pas).
- **Ne pas copier tout le catalogue vendor** dans le fichier applicatif. Seule la clé override est nécessaire. Laisser le reste venir du vendor évite de rater les nouvelles traductions ajoutées aux updates Sylius.
- **Apostrophes / caractères spéciaux** : quoter la valeur si elle contient `:`, `#`, ou commence par `-` / `?`. YAML peut sinon mal parser :
  ```yaml
  some_key: "Ajouter au panier : c'est parti"
  ```
- **Placeholders Symfony** (`{{ limit }}`, `%count%`) : conserver **exactement** les mêmes placeholders que la version vendor — sinon la chaîne s'affiche avec le placeholder brut.

### 4 — Cas domaine `validators`

Exemple : override du message « This value is too short » sur la longueur min d'un `Product.name`.

```yaml
# translations/validators.en.yaml

sylius:
    product:
        name:
            min_length: 'The product name is too short. It must be at least {{ limit }} characters long.'
```

La clé doit correspondre à ce qui est déclaré dans la contrainte côté `config/validator/ProductTranslation.yaml`. Si tu viens de créer la contrainte avec **`/sylius:validation`**, la clé est probablement `app.product.name.min_length` — mets-la dans le même domaine `validators.<locale>.yaml`.

### 5 — Cas domaine `flashes`

Les messages flash Sylius suivent la convention `sylius.<resource>.<action>` (ex. `sylius.product.create`, `sylius.customer.update`). Override :

```yaml
# translations/flashes.en.yaml

sylius:
    product:
        create: Product successfully registered!
```

Pour vérifier la clé exacte, inspecter la réponse HTTP après l'action ou grep `FlashHelper` / `FlashBag` dans le controller concerné.

### 6 — Override depuis un plugin Sylius

Si tu livres la customisation dans un plugin (pas dans `config/` applicatif) :

```
src/MyPluginBundle/Resources/translations/messages.en.yaml
src/MyPluginBundle/Resources/translations/messages.fr.yaml
```

- Le Translator charge automatiquement les catalogues d'un bundle enregistré. Pas de configuration supplémentaire nécessaire.
- L'application peut toujours re-override : si `my_plugin` change `sylius.form.customer.email: "Login"` et que l'app veut `"User ID"`, écrire `sylius.form.customer.email: "User ID"` dans `translations/messages.en.yaml` (applicatif) suffit. L'app gagne.

### 7 — Vérifier

```bash
# Vider le cache (obligatoire en prod, recommandé en dev)
php bin/console cache:clear

# Recharger la page dans le navigateur et confirmer le nouveau libellé
# Optionnel : dumper tous les catalogues pour la locale
php bin/console debug:translation en
```

Si le libellé n'a pas changé :

- la clé ou le domaine est faux → vérifier via le Profiler, pas deviner ;
- l'indentation YAML diverge du vendor → la clé ne matche pas ;
- mauvaise locale visitée → l'override ciblait `fr`, mais le visiteur est en `en` ;
- cache non vidé en `prod` → `cache:clear` puis `cache:warmup` si déploiement ;
- le texte vient d'un template Twig hardcodé (pas un `{{ 'sylius.form...' | trans }}`) → l'override YAML ne peut rien, il faut override le template via **`/sylius:template`**.

### 8 — Clôture

Afficher :

- **Fichiers créés/modifiés** : `translations/<domaine>.<locale>.yaml` pour chaque locale couverte.
- **Clé(s) override** et domaine(s).
- **Ce qui reste** : couvrir les autres locales actives, répercuter côté admin **et** shop si la clé est partagée, pousser l'override dans le plugin ou le garder applicatif, ajouter un test smoke (ouvrir la page et asserter le libellé) si la chaîne est critique.

## Pièges fréquents

- **Mauvais domaine** : déclarer `sylius.form.customer.email: Username` dans `validators.en.yaml`. Le form lit `messages` par défaut → la chaîne vendor reste affichée. Toujours confirmer le domaine via le Profiler.
- **Clé plate au lieu d'imbriquée** : écrire `sylius.form.customer.email: Username` comme clé littérale (avec les points). YAML stocke alors la clé entière comme chaîne et Symfony ne la matche pas. Respecter l'arborescence YAML ou préfixer la clé plate avec `!` / `''` selon le format attendu (le YAML imbriqué est toujours plus sûr).
- **Override partiel sur une seule locale** : les visiteurs `fr` voient le libellé vendor, les visiteurs `en` le nouveau. Soit assumé (locale-specific branding), soit bug — toujours lister les locales actives avant de boucler.
- **Placeholder supprimé** : retirer `{{ limit }}` du `minMessage` → la chaîne s'affiche avec le nombre manquant, ou pire, le placeholder littéral `{{ limit }}`. Toujours conserver les mêmes placeholders que le vendor.
- **Modifier le fichier `vendor/sylius/…/translations/*.yaml`** directement : saute au `composer update` suivant. Toujours écrire côté `translations/` applicatif ou plugin.
- **Template Twig avec texte hardcodé** : `<button>Add to cart</button>` au lieu de `<button>{{ 'sylius.ui.add_to_cart' | trans }}</button>`. Aucun override YAML ne changera ça — passer par `/sylius:template` (Twig hook ou override `templates/bundles/`).
- **Cache non vidé en `prod`** : la modif reste invisible après déploiement. `cache:clear` + `cache:warmup` systématiques.
- **Conflit app vs plugin** : l'app et un plugin custom déclarent la même clé avec des valeurs différentes. L'app gagne (ordre de priorité) — le mainteneur du plugin ne comprend pas pourquoi son libellé ne passe pas. Documenter l'override applicatif pour éviter la confusion.

## Argument optionnel

`/sylius:translation sylius.form.customer.email Username en` — override de la clé `sylius.form.customer.email` en anglais, domaine `messages` par défaut.

`/sylius:translation sylius.ui.add_to_cart "Buy now" en,fr` — override sur plusieurs locales en une passe.

`/sylius:translation sylius.product.name.min_length "..." en --domain=validators` — override dans le domaine `validators`.

`/sylius:translation` sans argument — demande la clé, la valeur, les locales, le domaine, et guide la détection via Symfony Profiler.

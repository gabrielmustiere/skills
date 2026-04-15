---
name: translation-entity
description: Crée une entité Sylius traduisible (pattern personal translations) : AbstractTranslation, TranslatableInterface, TranslatableTrait, locale fallback, ajout programmatique. Pour des libellés UI statiques → `/sylius:translation`.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /translation-entity — Entité traduisible Sylius

Tu aides à créer ou compléter une entité Sylius traduisible en suivant le pattern **personal translations** : chaque entité traduisible est liée à une entité `*Translation` dédiée avec sa propre table (ex. `Product` ↔ `ProductTranslation` / `sylius_product_translation`), au lieu d'une table fourre-tout pour tout le système.

Référence officielle : [docs.sylius.com/the-book/architecture/translations](https://docs.sylius.com/the-book/architecture/translations).

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine.
2. Vérifier `sylius/sylius` (ou au moins `sylius/resource-bundle`) dans les dépendances.
   - Présent → OK.
   - Absent → *« Ce skill cible Sylius (personal translations via ResourceBundle). Je ne trouve pas `sylius/sylius` ni `sylius/resource-bundle`. On continue quand même ou on bascule sur une approche Doctrine générique (ex. KNPLabs/DoctrineBehaviors Translatable) ? »*
3. Pour la création de l'entité elle-même, préférer la skill plus générique **`/symfony:doctrine-entity`** — cette skill traite spécifiquement la **paire traduisible + translation**.

## Règles fondamentales

- **Deux classes, pas une** : l'entité principale (`Supplier`) + l'entité de traduction (`SupplierTranslation`). Les champs traduisibles vivent sur la translation ; l'entité principale n'expose que des proxies via `getTranslation()`.
- **`SupplierTranslation extends AbstractTranslation`** : hérite de `Sylius\Component\Resource\Model\AbstractTranslation` qui fournit déjà `id`, `locale`, et la relation `translatable` vers l'entité parente.
- **`Supplier implements TranslatableInterface` + `use TranslatableTrait`** : le trait fournit `getTranslation()`, `setFallbackLocale()`, `addTranslation()`, `removeTranslation()`, `hasTranslation()`.
- **Initialiser la collection de traductions dans le constructeur** — sinon `addTranslation()` explose sur `null`. Utiliser l'alias `__construct as private initializeTranslationsCollection` du trait pour pouvoir ajouter sa propre logique dans le constructeur.
- **Proxifier les getters/setters** des champs traduisibles sur l'entité principale : `$this->getTranslation()->getName()`, `$this->getTranslation()->setName($name)`. Ça permet au code appelant d'ignorer la mécanique de traduction.
- **Table `sylius_<resource>_translation`** : respecter la convention de nommage via `#[ORM\Table(name: 'sylius_supplier_translation')]`, cohérent avec le reste du vendor.
- **Locale courante vs forcée** : `getTranslation()` sans argument utilise la locale courante de la requête ; `getTranslation('pl_PL')` force explicitement une locale.
- **Fallback locale** : si la traduction demandée n'existe pas, Sylius retombe sur la `fallback_locale` configurée dans `config/services.yaml` ou via `setFallbackLocale()` sur l'entité. Ne jamais renvoyer `null` silencieusement depuis un getter proxifié — laisser le trait gérer.
- **Validation** : les contraintes sur les champs traduisibles vont sur la classe `*Translation`, pas sur l'entité principale. Utiliser `groups: ['Default', 'sylius']` pour que les forms Sylius les exécutent.

## Déroulement

### 1 — Cadrer le besoin

Demander (ou confirmer si déjà fourni) :

- Nom de l'entité principale (PascalCase, singulier) — ex. `Supplier`, `Product`, `Brand`.
- Champs **traduisibles** (vont sur `*Translation`) : nom, description, slug, meta tags, etc.
- Champs **non traduisibles** (restent sur l'entité principale) : code, prix, dates, statuts, relations métier.
- Héritage : étend-on une entité Sylius native (`Product`, `Taxon`, etc.) ou on crée une entité maison from scratch ?

### 2 — Générer les deux classes

Passer par **`/symfony:doctrine-entity`** pour l'entité principale et l'entité de traduction, puis revenir ici pour câbler le pattern translatable. Ne pas réécrire à la main ce que `make:entity` sait générer.

### 3 — Câbler la translation

**Entité de traduction** — `src/Entity/Supplier/SupplierTranslation.php` :

```php
<?php

declare(strict_types=1);

namespace App\Entity\Supplier;

use Doctrine\ORM\Mapping as ORM;
use Sylius\Component\Resource\Model\AbstractTranslation;

#[ORM\Entity]
#[ORM\Table(name: 'app_supplier_translation')]
class SupplierTranslation extends AbstractTranslation
{
    #[ORM\Id]
    #[ORM\Column(type: 'integer')]
    #[ORM\GeneratedValue(strategy: 'AUTO')]
    private ?int $id = null;

    #[ORM\Column(type: 'string', length: 255)]
    private ?string $name = null;

    public function getId(): ?int
    {
        return $this->id;
    }

    public function getName(): ?string
    {
        return $this->name;
    }

    public function setName(?string $name): void
    {
        $this->name = $name;
    }
}
```

**Entité principale** — `src/Entity/Supplier/Supplier.php` :

```php
<?php

declare(strict_types=1);

namespace App\Entity\Supplier;

use Doctrine\ORM\Mapping as ORM;
use Sylius\Component\Resource\Model\TranslatableInterface;
use Sylius\Component\Resource\Model\TranslatableTrait;

#[ORM\Entity]
#[ORM\Table(name: 'app_supplier')]
class Supplier implements TranslatableInterface
{
    use TranslatableTrait {
        __construct as private initializeTranslationsCollection;
    }

    #[ORM\Id]
    #[ORM\Column(type: 'integer')]
    #[ORM\GeneratedValue(strategy: 'AUTO')]
    private ?int $id = null;

    #[ORM\Column(type: 'string', length: 64, unique: true)]
    private ?string $code = null;

    public function __construct()
    {
        $this->initializeTranslationsCollection();
    }

    public function getId(): ?int
    {
        return $this->id;
    }

    public function getCode(): ?string
    {
        return $this->code;
    }

    public function setCode(?string $code): void
    {
        $this->code = $code;
    }

    public function getName(): ?string
    {
        return $this->getTranslation()->getName();
    }

    public function setName(?string $name): void
    {
        $this->getTranslation()->setName($name);
    }

    protected function createTranslation(): SupplierTranslation
    {
        return new SupplierTranslation();
    }
}
```

Points de vigilance :

- **`createTranslation()`** est nécessaire pour que `getTranslation()` puisse instancier une nouvelle translation à la volée quand la locale courante n'en a pas encore.
- **Garder le getter/setter proxy** aligné avec le champ réel de la translation (même nullabilité, même type).
- **Pas de logique métier** dans le getter proxifié — laisser la translation porter la règle.

### 4 — Déclarer la resource Sylius

Dans `config/packages/_sylius.yaml`, déclarer la paire :

```yaml
sylius_resource:
    resources:
        app.supplier:
            classes:
                model: App\Entity\Supplier\Supplier
            translation:
                classes:
                    model: App\Entity\Supplier\SupplierTranslation
```

Si on **étend** une resource Sylius existante (`sylius.product`, `sylius.taxon`…), remplacer `model` sous `sylius_<bundle>.resources.<resource>.classes.model` et `…translation.classes.model`. Ne pas créer une nouvelle resource à côté.

### 5 — Migration et validation

```bash
symfony console doctrine:schema:validate
```

Puis générer la migration via **`/symfony:doctrine-migration`** (ne pas lancer `make:migration` depuis ici).

### 6 — Ajout programmatique de traductions

Pattern pour seeder ou créer une traduction à la volée :

```php
/** @var ProductInterface $product */
$product = $this->container->get('sylius.repository.product')->findOneBy(['code' => 'radiohead-mug']);

/** @var ProductTranslation $translation */
$translation = new ProductTranslation();
$translation->setLocale('pl_PL');
$translation->setName('Kubek Radiohead');
$translation->setSlug('kubek-radiohead');

$product->addTranslation($translation);

$this->container->get('sylius.manager.product')->flush();
```

- **`setLocale()`** est obligatoire — sans locale, la translation est orpheline.
- **`addTranslation()`** met aussi à jour `translation->translatable` via le trait.
- **Ne pas oublier `flush()`** sur le manager approprié.

### 7 — Fallback locale

Configuré globalement dans `config/services.yaml` :

```yaml
parameters:
    locale: en_US
    app.fallback_locale: en_US
```

Surchargeable par entité via `$entity->setFallbackLocale('fr_FR')` si besoin d'un comportement spécifique (rare — la config globale suffit dans 95 % des cas).

### 8 — Clôture

Afficher :

- Fichiers créés/modifiés (`Supplier.php`, `SupplierTranslation.php`, `_sylius.yaml`).
- Champs traduisibles vs non traduisibles.
- Ce qui reste : migration (`/symfony:doctrine-migration`), form type avec `ResourceTranslationsType` (voir plugin workflow), fixtures multi-locale, mise à jour du ProductRepository si filtre par nom traduit.

## Pièges fréquents

- **`getTranslation()` retourne toujours une instance** (jamais `null`) grâce à `createTranslation()` — ne pas tester `if ($translation === null)`, tester plutôt `$translation->getName() === null`.
- **Champ traduisible oublié sur la form** : si le form ne liste pas le champ dans `ResourceTranslationsType`, il sera perdu au submit. Utiliser un `FormTypeExtension` pour les entités natives Sylius qu'on étend.
- **Recherche SQL sur un champ traduisible** : ne pas `WHERE supplier.name = ...`, mais joindre la table de traduction : `JOIN supplier.translations translation WITH translation.locale = :locale WHERE translation.name = :name`.
- **Doublon de translation par locale** : le trait ne pose pas de contrainte unique sur `(translatable_id, locale)` par défaut — ajouter `#[ORM\UniqueConstraint(columns: ['translatable_id', 'locale'])]` sur `*Translation` si on veut garantir l'unicité au niveau DB.
- **Héritage Sylius** : si on étend `Sylius\Component\Core\Model\Product`, ne pas re-déclarer `TranslatableTrait` — la classe de base l'utilise déjà. Étendre plutôt `ProductTranslation` pour ajouter des champs traduisibles custom.

## Argument optionnel

`/sylius:translation-entity Supplier` — cadre et génère la paire `Supplier` / `SupplierTranslation`.

`/sylius:translation-entity src/Entity/Product/Product.php` — audite une entité existante et ajoute le pattern translatable si manquant, ou complète la translation associée.

`/sylius:translation-entity` sans argument — demande le nom de l'entité et les champs traduisibles.

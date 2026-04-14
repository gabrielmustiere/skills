---
name: form-advanced
description: Scénarios avancés Symfony forms — DataTransformer, FormEvents, CollectionType, FileType, CSRF, FormTypeExtension. Déclenche sur "DataTransformer", "FormEvents", "form dynamique", "CollectionType", "FileType", "VichUploader".
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /form-advanced — Scénarios avancés Symfony forms

Tu gères les cas qui dépassent le `FormType` de base : transformer les données entrantes/sortantes, modifier le form selon le contexte, imbriquer une collection, uploader un fichier, gérer le CSRF finement.

## Détection préalable

Lire `composer.json` ; vérifier `symfony/framework-bundle` + `symfony/form`. Pour upload : vérifier `vich/uploader-bundle` (recommandé) ou implémenter à la main. Mentionner Sylius si présent (Sylius a ses propres CollectionType pour images, translations — utiliser les resources Sylius plutôt que recoder).

## 1 — DataTransformer

**Quand** : le format côté objet diffère du format côté formulaire. Exemples : tag `string "php,symfony"` côté form / `Tag[]` côté entité ; ID numérique dans une query string / entité côté contrôleur ; date string `"2026-04-14"` / `DateTimeImmutable`.

**Interface** : `DataTransformerInterface` avec `transform($value)` (model → view) et `reverseTransform($value)` (view → model).

```php
final class TagArrayToStringTransformer implements DataTransformerInterface
{
    public function __construct(private readonly TagRepository $tags) {}

    public function transform($tags): string
    {
        return $tags === null ? '' : implode(',', array_map(fn (Tag $t) => $t->getName(), $tags));
    }

    public function reverseTransform($value): array
    {
        if ($value === null || $value === '') {
            return [];
        }
        $names = array_filter(array_map('trim', explode(',', $value)));
        return array_map(fn (string $n) => $this->tags->findOrCreate($n), $names);
    }
}
```

Attaché dans le FormType :

```php
$builder->get('tags')->addModelTransformer(new TagArrayToStringTransformer($this->tags));
```

**Règles** :

- Lever `TransformationFailedException` sur donnée invalide — Symfony l'affiche comme erreur sur le champ.
- `addModelTransformer` (model ↔ norm) vs `addViewTransformer` (norm ↔ view) : dans 90 % des cas, model transformer suffit.
- Un transformer **ne valide pas** la donnée métier : Assert fait ça. Le transformer ne fait que changer le format.

## 2 — FormEvents (forms dynamiques)

**Quand** : le form change selon l'objet ou la soumission. Exemples : un `ChoiceType` de ville dépend du pays sélectionné ; un champ n'apparaît qu'en mode création ; un champ dérivé est recalculé à la soumission.

**Événements clés** :

| Event            | Déclenché                                    | Usage                                          |
|------------------|----------------------------------------------|------------------------------------------------|
| `PRE_SET_DATA`   | avant que les données initiales soient posées | ajouter/retirer un champ selon l'objet          |
| `POST_SET_DATA`  | après                                        | rarement utile                                  |
| `PRE_SUBMIT`     | données brutes arrivées, avant transformation | modifier des champs selon la soumission (ex: dépendance pays→ville) |
| `SUBMIT`         | données validées, avant écriture dans l'objet | ajustements finaux                             |
| `POST_SUBMIT`    | après écriture                               | validation croisée                              |

```php
$builder->addEventListener(FormEvents::PRE_SET_DATA, function (FormEvent $event): void {
    $product = $event->getData();
    $form    = $event->getForm();

    if ($product instanceof Product && $product->getId() !== null) {
        // édition : SKU verrouillé
        $form->add('sku', TextType::class, ['disabled' => true]);
    } else {
        // création : SKU libre
        $form->add('sku', TextType::class);
    }
});

$builder->get('country')->addEventListener(FormEvents::POST_SUBMIT, function (FormEvent $event): void {
    $country = $event->getForm()->getData();
    $form    = $event->getForm()->getParent();

    $form->add('city', EntityType::class, [
        'class' => City::class,
        'choices' => $this->cities->findByCountry($country),
    ]);
});
```

**Règles** :

- `disabled: true` est **plus sûr** qu'une omission de champ quand le champ doit rester affiché mais non modifiable — Symfony ignore la valeur soumise.
- Dépendance champ A → champ B : écouter `POST_SUBMIT` sur A pour reconstruire B. Pour le GET initial, même logique via `PRE_SET_DATA`.
- Pas de requête BDD lourde dans un event : injecter un service et cacher si besoin.

## 3 — CollectionType (forms imbriqués)

**Quand** : une entité a une relation OneToMany/ManyToMany modifiable dans le même form. Exemples : `Order` → `OrderItem[]`, `Product` → `Tag[]`, `Survey` → `Question[]`.

```php
// ProductType
->add('variants', CollectionType::class, [
    'entry_type' => VariantType::class,
    'entry_options' => ['label' => false],
    'allow_add'     => true,
    'allow_delete'  => true,
    'by_reference'  => false,
    'prototype'     => true,
])
```

**Options critiques** :

- `allow_add` / `allow_delete` — active l'ajout/suppression côté JS. Sans ça, seul le nombre initial de lignes est modifiable.
- `by_reference: false` — **obligatoire** pour que les setters de l'entité parent soient appelés (sinon les enfants sont modifiés en place mais Doctrine ne "voit" pas le parent mis à jour). À mettre même sur une collection qui n'accepte ni add ni delete si le parent a des callbacks `addVariant`/`removeVariant`.
- `prototype: true` — Symfony injecte une ligne `<div data-prototype="…">` dont le JS côté client s'inspire pour ajouter dynamiquement des éléments.

**Côté entité parent**, implémenter `addX` / `removeX` qui gèrent la réciprocité :

```php
public function addVariant(Variant $variant): self
{
    if (!$this->variants->contains($variant)) {
        $this->variants->add($variant);
        $variant->setProduct($this);
    }
    return $this;
}

public function removeVariant(Variant $variant): self
{
    if ($this->variants->removeElement($variant)) {
        if ($variant->getProduct() === $this) {
            $variant->setProduct(null);
        }
    }
    return $this;
}
```

**Côté Twig**, rendre le prototype :

```twig
<div class="variants" data-prototype="{{ form_widget(form.variants.vars.prototype)|e('html_attr') }}">
    {% for variant in form.variants %}
        {{ form_row(variant) }}
    {% endfor %}
</div>
```

Côté JS (vanilla ou Stimulus), cloner le `data-prototype` en remplaçant `__name__` par un index incrémenté.

**Cascade persist** sur la relation (`cascade: ['persist']` côté ORM) sinon les enfants ajoutés ne sont jamais sauvés.

## 4 — FileType (upload de fichier)

**Cas simple (une image, sans bundle)** :

```php
->add('photoFile', FileType::class, [
    'mapped'      => false,
    'required'    => false,
    'constraints' => [
        new File(maxSize: '5M', mimeTypes: ['image/png', 'image/jpeg']),
    ],
])
```

Dans le contrôleur, après `isValid()` :

```php
$uploaded = $form->get('photoFile')->getData();
if ($uploaded instanceof UploadedFile) {
    $filename = $uploader->store($uploaded, 'products');   // service dédié
    $product->setPhotoPath($filename);
}
```

Le service `Uploader` appelle `$uploaded->move($targetDir, $newFilename)` avec un nom généré (`bin2hex(random_bytes(16)) . '.' . $uploaded->guessExtension()`) pour éviter les collisions et le filename injection.

**Cas réaliste (multi-upload, versions, associations)** : utiliser **VichUploaderBundle**. Il gère le stockage, le nom, la suppression sur delete, les namers. Mapping via attributs `#[Vich\Uploadable]` + `#[Vich\UploadableField]` sur l'entité.

**Règles** :

- **Jamais de chemin utilisateur** dans le filename final — toujours regénérer.
- **Valider la taille et le mime-type** via `Assert\File`. Le `mimeTypes` est vérifié via `finfo`, pas seulement l'extension.
- **Stocker hors du dossier public** si le fichier ne doit pas être téléchargeable directement ; servir via un contrôleur qui check les droits.

## 5 — CSRF

- **Activé par défaut** sur tous les forms HTML. Ne pas désactiver, sauf pour une API JSON qui utilise un autre mécanisme d'auth (bearer token, same-site cookie + CORS).
- **Désactivation locale** (ex: un form purement GET de recherche) :

```php
// configureOptions
$resolver->setDefaults([
    'csrf_protection' => false,
]);
```

- **Token name custom** :

```php
$resolver->setDefaults([
    'csrf_field_name' => '_token',
    'csrf_token_id'   => 'product_edit',   // par défaut = nom du form
]);
```

- Pour une action **hors form** (bouton "supprimer"), générer un token manuellement :

```twig
<form method="POST" action="{{ path('product_delete', {id: product.id}) }}">
    <input type="hidden" name="_token" value="{{ csrf_token('delete-product-' ~ product.id) }}">
    <button>Supprimer</button>
</form>
```

Côté contrôleur : `$this->isCsrfTokenValid('delete-product-'.$product->getId(), $request->request->get('_token'))`.

## 6 — FormTypeExtension

**Quand** : on veut ajouter un comportement à **tous les types** (ou à une sous-famille). Exemple : ajouter automatiquement une classe CSS `form-control` sur tous les `TextType`, injecter un help text depuis un traducteur.

```php
final class HelpLinkExtension extends AbstractTypeExtension
{
    public static function getExtendedTypes(): iterable
    {
        return [FormType::class];   // s'applique à tous
    }

    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefault('help_link', null);
    }

    public function buildView(FormView $view, FormInterface $form, array $options): void
    {
        $view->vars['help_link'] = $options['help_link'];
    }
}
```

Autowiring l'enregistre automatiquement via le tag `form.type_extension`.

## Delta Sylius

- Images de produit → `ProductImage` + `ImageUploadType` Sylius (ne pas recoder un upload manuel).
- Traductions → `ResourceTranslationsType` Sylius, pas un `CollectionType` maison.
- `CollectionType` de resources Sylius surchargées → garder `by_reference: false` et vérifier que la resource custom a bien les `add*`/`remove*` avec la réciprocité.

## Argument optionnel

`/symfony:form-advanced transformer TagType tags` — ajoute un `DataTransformer` sur le champ `tags` de `TagType`.

`/symfony:form-advanced collection ProductType variants` — ajoute un `CollectionType` de variantes dans `ProductType` + pattern JS prototype.

`/symfony:form-advanced upload ProductType photoFile` — ajoute un `FileType` + service d'upload.

`/symfony:form-advanced events ProductType` — ajoute des FormEvents (PRE_SET_DATA mode création/édition, dépendance pays/ville…).

`/symfony:form-advanced` sans argument — demande le scénario et le FormType cible.

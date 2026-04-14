---
name: form-handle
description: Traite la soumission d'un formulaire Symfony — createForm, handleRequest, isSubmitted/isValid, PRG. Déclenche sur "traiter formulaire", "handleRequest", "isSubmitted && isValid", "redirection après form". Impose délégation métier et redirection.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /form-handle — Traitement d'un formulaire dans un contrôleur

Tu écris ou révises l'action contrôleur qui reçoit un `FormType`. Tu appliques le pattern canonique (`handleRequest` + `isSubmitted && isValid` + redirection) et tu ne laisses **aucune logique métier** dans le contrôleur.

## Détection préalable

Lire `composer.json`, vérifier `symfony/framework-bundle` + `symfony/form`. Mentionner Sylius en une ligne si présent (contrôleurs Sylius ont souvent un `ResourceController` générique — ne pas réécrire le CRUD Sylius, surcharger la resource).

## Règles fondamentales

- **Pattern canonique** : `createForm` → `handleRequest($request)` → `if ($form->isSubmitted() && $form->isValid())` → persist + flush → `redirectToRoute` (PRG, Post-Redirect-Get). Afficher le template dans tous les autres cas (GET initial, soumission invalide).
- **`handleRequest` > `submit`** sauf pour API JSON avec payload non-form-encoded. `handleRequest` gère la méthode HTTP, les fichiers, le token CSRF, les boutons cliqués.
- **Logique métier déléguée** à un service (`ProductCreator`, `OrderCompleter`, command handler). Le contrôleur orchestre : form → service → flash → redirect. Pas d'appel à l'EntityManager dans le contrôleur au-delà de ce qu'impose le form si tu utilises le pattern CQRS.
- **Pattern PRG obligatoire** : toute soumission valide se termine par une redirection. Sinon un F5 resoumet le form — double création, paiement doublé, etc.
- **Message flash** sur succès et sur échec (attendu par le template), pas de retour texte brut.
- **`$form->createView()` n'est plus nécessaire** depuis Symfony 6.2 : passer `$form` directement au template, Twig appelle `createView()` implicitement. Ne pas remettre `createView()` dans le contrôleur sauf version antérieure.
- **Ne jamais appeler `$form->getData()` avant `isSubmitted() && isValid()`** pour construire un comportement métier — les données ne sont fiables qu'après validation.

## Squelette canonique

```php
#[Route('/product/new', name: 'product_new', methods: ['GET', 'POST'])]
public function new(
    Request $request,
    ProductCreator $creator,
): Response {
    $product = new Product();
    $form = $this->createForm(ProductType::class, $product);
    $form->handleRequest($request);

    if ($form->isSubmitted() && $form->isValid()) {
        $creator->create($product);
        $this->addFlash('success', 'product.created');

        return $this->redirectToRoute('product_show', ['id' => $product->getId()]);
    }

    return $this->render('product/new.html.twig', [
        'form' => $form,
    ]);
}
```

## Cas particuliers

### Édition

Même pattern, l'objet vient de la BDD :

```php
public function edit(Request $request, Product $product, ProductUpdater $updater): Response
{
    $form = $this->createForm(ProductType::class, $product);
    $form->handleRequest($request);

    if ($form->isSubmitted() && $form->isValid()) {
        $updater->update($product);
        $this->addFlash('success', 'product.updated');

        return $this->redirectToRoute('product_show', ['id' => $product->getId()]);
    }

    return $this->render('product/edit.html.twig', ['form' => $form, 'product' => $product]);
}
```

`#[MapEntity]` implicite via le type-hint `Product $product` — voir `/symfony:doctrine-query` pour les cas avec slug ou filtre custom.

### Boutons multiples

```php
// Dans ProductType::buildForm
$builder
    ->add('save', SubmitType::class, ['label' => 'Enregistrer'])
    ->add('saveAndAdd', SubmitType::class, ['label' => 'Enregistrer et ajouter']);
```

```php
if ($form->isSubmitted() && $form->isValid()) {
    $creator->create($product);

    if ($form->get('saveAndAdd')->isClicked()) {
        return $this->redirectToRoute('product_new');
    }

    return $this->redirectToRoute('product_show', ['id' => $product->getId()]);
}
```

### Options custom

```php
$form = $this->createForm(ProductType::class, $product, [
    'require_sku' => $product->getId() !== null,    // édition stricte, création souple
    'current_user' => $this->getUser(),
]);
```

### Form sans `data_class`

Pour un form hors entité (filtre, recherche, import CSV) :

```php
$form = $this->createFormBuilder()
    ->add('query', SearchType::class)
    ->add('category', EntityType::class, ['class' => Category::class])
    ->getForm();

$form->handleRequest($request);
if ($form->isSubmitted() && $form->isValid()) {
    $data = $form->getData(); // tableau associatif
    $results = $searcher->search($data['query'], $data['category']);
}
```

Préférer tout de même un `FormType` nommé (`ProductSearchType`) pour toute recherche non-triviale.

### Champs non mappés (`getExtraData`)

```php
$rawCaptcha = $form->get('captcha')->getData();   // champ mapped: false
$extras     = $form->getExtraData();              // champs hors FormType si allow_extra_fields
```

### Erreurs globales (non-field)

Si la validation échoue au niveau d'une contrainte de classe (Unique, Callback), l'erreur est sur le form racine :

```twig
{{ form_errors(form) }}
```

Et côté contrôleur pas de traitement spécial — le `isValid()` renvoie `false`, le template affiche.

## Tests

Un test functional par action de form :

```php
public function testSubmitValidProduct(): void
{
    $client = static::createClient();
    $client->request('GET', '/product/new');
    $client->submitForm('Enregistrer', [
        'product[name]' => 'Widget',
        'product[priceCents]' => '1999',
    ]);

    self::assertResponseRedirects();
    self::assertSame(1, $this->productRepository->count([]));
}
```

Pour les FormType pris isolément → `Symfony\Component\Form\Test\TypeTestCase` (ou `FormTestCase` en 7.x).

## Delta Sylius

- Les CRUD resources Sylius passent par `ResourceController` — **ne pas recoder** un contrôleur. Surcharger la config de resource (`_sylius.yaml`), le form (`/symfony:form-type`), le template si besoin (`/symfony:form-render`).
- Events de cycle de vie Sylius (`sylius.product.pre_create`, `post_update`) → préférer un event subscriber à un contrôleur custom.

## Argument optionnel

`/symfony:form-handle ProductController new` — écrit/révise l'action `new` qui traite `ProductType`.

`/symfony:form-handle src/Controller/OrderController.php` — audit de tout le fichier (pattern PRG, isValid manquant, logique métier dans le contrôleur).

`/symfony:form-handle` sans argument — demande le contrôleur, l'action et le FormType.

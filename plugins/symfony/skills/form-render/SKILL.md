---
name: form-render
description: Rend un formulaire Symfony dans Twig — form_row, form_widget, form_errors, thèmes (bootstrap_5, tailwind). Déclenche sur "rendu form Twig", "form_row", "form_theme". Impose thème global et form_row.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /form-render — Rendu Twig d'un formulaire Symfony

Tu écris ou révises le template qui affiche un form. Tu privilégies `form_row` et un thème global, tu ne casses la granularité qu'en cas de besoin réel (layout complexe, champs côte à côte, widgets intercalés).

## Détection préalable

Lire `composer.json` ; vérifier `symfony/twig-bundle` et `symfony/form`. Mentionner Sylius en une ligne si présent (Sylius a ses propres thèmes de form côté admin/shop — les templates de resource sont surchargés via `@SyliusAdmin` ou `@SyliusShop`).

## Règles fondamentales

- **Thème défini globalement** dans `config/packages/twig.yaml` sous `form_themes`. Ne pas mettre `{% form_theme form 'bootstrap_5_layout.html.twig' %}` dans chaque template — c'est fait une fois, partout.
- **`form_row` par défaut**. Rendre `form_label` + `form_widget` + `form_errors` manuellement est réservé aux layouts qui l'exigent (form horizontal custom, widgets côte à côte, texte intercalé).
- **`{{ form(form) }}` est acceptable** quand le form est simple et n'a besoin d'aucun HTML autour des champs. Dès qu'un `<fieldset>`, un séparateur, ou deux champs sur une même ligne entre en jeu → passer à la forme explicite avec `form_start` / `form_end`.
- **`form_end` appelle `form_rest`** qui rend les champs oubliés + le CSRF. Ne jamais désactiver `render_rest: false` sauf nécessité absolue (API-like avec CSRF désactivé).
- **`novalidate`** activé sur la form pendant le développement pour tester la validation serveur ; retirer en prod si la validation HTML5 est acceptable.
- **Attributs via `attr`** (côté FormType ou dans `form_widget(…, {'attr': {…}})`), pas de concaténation HTML dans Twig.
- **Customisation par bloc Twig**, pas par surcharge HTML à la main. Le bloc `{% block _<form_name>_<field>_widget %}` surcharge un champ précis ; `{% block form_row %}` dans un thème local surcharge tous les rows.

## Configuration globale

```yaml
# config/packages/twig.yaml
twig:
    form_themes:
        - 'bootstrap_5_layout.html.twig'
        # - 'bootstrap_5_horizontal_layout.html.twig'   # label à gauche du champ
        # - 'tailwind_2_layout.html.twig'
        # - 'foundation_6_layout.html.twig'
```

Ordre important : les thèmes de **bas** en **haut** s'appliquent en cascade — le plus haut prime. Pour une surcharge projet, ajouter un fichier maison en bas (ex: `form/_project.html.twig`) qui étend `bootstrap_5_layout.html.twig`.

## Patterns de rendu

### Rendu basique (form simple)

```twig
{{ form_start(form) }}
    {{ form_widget(form) }}
    <button type="submit" class="btn btn-primary">Enregistrer</button>
{{ form_end(form) }}
```

### Rendu granulaire (layout custom)

```twig
{{ form_start(form, {'attr': {'novalidate': 'novalidate'}}) }}
    <div class="row">
        <div class="col-md-6">
            {{ form_row(form.firstName) }}
        </div>
        <div class="col-md-6">
            {{ form_row(form.lastName) }}
        </div>
    </div>

    <fieldset>
        <legend>Adresse</legend>
        {{ form_row(form.address.street) }}
        {{ form_row(form.address.city) }}
    </fieldset>

    {{ form_rest(form) }}

    <button type="submit" class="btn btn-primary">Enregistrer</button>
{{ form_end(form) }}
```

`form_rest` rend **tout ce qui n'a pas encore été rendu** explicitement, y compris le token CSRF. Le garder est une assurance contre l'oubli d'un champ.

### Passer des attributs HTML ponctuels

```twig
{{ form_row(form.email, {
    'attr': {'autocomplete': 'email', 'placeholder': 'vous@exemple.fr'},
    'label_attr': {'class': 'form-label-sm'},
    'help': 'Votre email professionnel',
}) }}
```

Pour des attributs **systématiques**, les définir dans le `FormType` :

```php
->add('email', EmailType::class, [
    'attr' => ['autocomplete' => 'email'],
])
```

### Changer action / méthode

```twig
{{ form_start(form, {
    'action': path('product_new'),
    'method': 'POST',
    'attr': {'class': 'my-form'},
}) }}
```

`method: 'PUT'|'PATCH'|'DELETE'` rend un `<form method="POST">` + champ caché `_method` — Symfony le traduit côté serveur si `framework.http_method_override: true`.

### Désactiver la validation HTML5

```twig
{{ form_start(form, {'attr': {'novalidate': 'novalidate'}}) }}
```

### Afficher les erreurs

```twig
{{ form_errors(form) }}               {# erreurs globales (contraintes de classe) #}
{{ form_errors(form.email) }}         {# erreurs du champ email (déjà inclus par form_row) #}
```

Ne jamais doubler : `form_row` inclut déjà `form_errors` du champ.

## Customisation par bloc

Dans un thème local `templates/form/_project.html.twig` :

```twig
{% use 'bootstrap_5_layout.html.twig' %}

{# Surcharge du widget de TOUS les champs money #}
{% block money_widget %}
    <div class="input-group">
        {{ parent() }}
        <span class="input-group-text">€</span>
    </div>
{% endblock %}
```

Pour un champ **unique** d'un form précis, cibler par le block prefix (nom du FormType + nom du champ) :

```twig
{% block _product_sku_widget %}
    <div class="font-monospace">
        {{ block('form_widget_simple') }}
    </div>
{% endblock %}
```

Le nom du block vient de `getBlockPrefix()` du FormType — par défaut dérivé du nom de la classe (`ProductType` → `product`). Surchargeable si besoin.

## Delta Sylius

- Les templates de resource (admin/shop) sont dans `@SyliusAdmin` / `@SyliusShop`. Surcharger via `templates/bundles/SyliusAdminBundle/…`.
- Sylius utilise ses propres thèmes (`@SyliusAdmin/Form/theme.html.twig`). Ajouter un thème projet **après** dans `twig.yaml` pour surcharger sans casser les widgets vendor.
- Champs traduisibles (`TranslationsType`) → le rendu standard fonctionne, mais le layout multi-onglets par locale vient du thème Sylius — ne pas le remplacer à la main.

## Déroulement

1. Identifier le template concerné (`templates/<resource>/<action>.html.twig`).
2. Vérifier `config/packages/twig.yaml` — le thème global est-il bien défini ?
3. Choisir le niveau de granularité : `{{ form(form) }}`, `form_widget`, ou `form_row` par champ.
4. Déplacer tout attribut systématique vers le `FormType` ; garder dans le template seulement ce qui est spécifique au template.
5. Si customisation de widget récurrente → créer/éditer le thème local (`templates/form/_project.html.twig`) et l'ajouter à `twig.yaml`.
6. Tester via le contrôleur (→ `/symfony:form-handle`).

## Argument optionnel

`/symfony:form-render templates/product/new.html.twig` — audit du template (granularité, attributs qui devraient migrer dans le FormType, thème).

`/symfony:form-render ProductType` — crée le template correspondant à un FormType existant.

`/symfony:form-render` sans argument — demande le template ou le FormType.

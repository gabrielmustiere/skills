---
name: form-type
description: Conçoit une classe FormType Symfony — AbstractType, buildForm, configureOptions, types de champs (ChoiceType, EntityType…). Déclenche sur "créer FormType", "buildForm", "data_class", "EntityType". Impose make:form.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /form-type — Conception d'un FormType Symfony

Tu conçois ou complètes une classe FormType (`src/Form/…Type.php`). Tu t'appuies sur `make:form`, tu choisis les bons types de champ, tu passes par `configureOptions` pour toute variante, et tu n'écris **jamais** un form inline dans un contrôleur.

## Détection préalable (obligatoire)

1. Lire `composer.json`.
2. Vérifier `symfony/framework-bundle` **et** `symfony/form`.
   - Présents → OK.
   - `symfony/form` absent → `composer require symfony/form` avant de continuer.
   - Stack non-Symfony → demander : *« Ce skill cible Symfony, je ne trouve pas les paquets attendus. On continue quand même ? »*
3. Si `sylius/sylius` présent → les forms Sylius héritent de `AbstractResourceType` et utilisent les groupes de validation `Default` + `sylius`. Détails dans `references/stacks/sylius.md` du plugin workflow.

## Règles fondamentales

- **`make:form` d'abord** : `symfony console make:form <Nom>Type <Entité>` génère le squelette mappé sur l'entité. Pas d'écriture manuelle complète.
- **Un FormType = un cas d'usage métier**. `ProductCreateType` et `ProductEditType` peuvent diverger (champs différents, validation différente) — ne pas forcer la réutilisation avec des options `mode=creation|edit` qui rendent le type illisible. Si 80 % des champs sont communs, extraire un parent via `getParent()`.
- **`data_class` toujours** sur un form mappé à une entité. Sans ça, Symfony ne peut pas deviner les types ni appliquer la validation de l'entité.
- **Types explicites** : `TextType::class`, `ChoiceType::class`, `EntityType::class`… Ne jamais passer `null` ou omettre le type **sauf** quand le type guessing est intentionnel (champ dérivé du mapping Doctrine sans option custom).
- **`property_path`** quand le nom du champ dans le form ne matche pas la propriété PHP (ex: champ `email` exposé mais propriété `username` côté entité).
- **`mapped: false`** pour les champs hors entité (captcha, confirmation de mot de passe en clair, acceptation des CGU). Ces champs sont lus via `$form->get('…')->getData()`, pas depuis l'objet.
- **Options custom** via `configureOptions` + `OptionsResolver` : chaque variante (`require_due_date`, `current_user`, etc.) est déclarée, typée, avec une valeur par défaut. Jamais de globale, jamais `$GLOBALS`.
- **Services injectables** via le constructeur (autowiring). Ne pas appeler le container dans `buildForm`.

## Types de champ les plus fréquents

| Besoin                              | Type                       | Options clés                                |
|-------------------------------------|----------------------------|---------------------------------------------|
| Texte court                         | `TextType`                 | `empty_data`, `trim`                        |
| Texte long                          | `TextareaType`             | `attr: {rows: 6}`                           |
| Email                               | `EmailType`                | validation `#[Assert\Email]` côté entité    |
| Nombre entier / décimal             | `IntegerType` / `NumberType` | `scale`, `html5: true`                    |
| Montant                             | `MoneyType`                | `currency`, `divisor: 100` (stockage cents) |
| Choix statique                      | `ChoiceType`               | `choices`, `expanded`, `multiple`           |
| Choix depuis une entité             | `EntityType`               | `class`, `choice_label`, `query_builder`    |
| Date / datetime                     | `DateType` / `DateTimeType`| `widget: 'single_text'`, `input: 'datetime_immutable'` |
| Booléen visible                     | `CheckboxType`             | `required: false` quasi systématique        |
| Fichier                             | `FileType`                 | cf. `/symfony:form-advanced`                |
| Sous-formulaire                     | `<Autre>Type::class`       | réutilisation d'un FormType                 |
| Collection de sous-formulaires      | `CollectionType`           | cf. `/symfony:form-advanced`                |
| Soumission                          | `SubmitType`               | à éviter dans le type — mettre dans le template |

`SubmitType` dans `buildForm` couple le form au template. Préférer `{{ form_widget(form) }}` puis un `<button>` séparé dans le template, sauf si le form a plusieurs boutons qui partagent la logique (ex: `save` + `saveAndAdd` — alors les mettre dans le type).

## Déroulement

### 1 — Cadrer

- Entité cible (ou form sans `data_class`).
- Cas d'usage : création, édition, filtre de recherche, action ponctuelle (reset password) ?
- Champs exposés, champs masqués, champs non mappés.
- Options variables entre contextes (`require_*`, utilisateur courant, channel Sylius).

### 2 — Générer

```bash
symfony console make:form <Nom>Type <Entité>   # entité optionnelle
```

### 3 — Compléter le générateur

Pour chaque champ :

- Type explicite (cf. tableau ci-dessus).
- Options : `label`, `help`, `required`, `empty_data`, `placeholder`, `attr: {…}`.
- `choices` ou `query_builder` pour les `ChoiceType` / `EntityType`. Pour `EntityType`, **toujours** un `query_builder` si la liste peut dépasser quelques dizaines de lignes — sinon toutes les lignes sont chargées.
- Contraintes **supplémentaires** via l'option `constraints: [...]` uniquement si elles sont propres au form (cf. `/symfony:doctrine-entity` pour la validation de l'entité).

### 4 — `configureOptions`

```php
public function configureOptions(OptionsResolver $resolver): void
{
    $resolver->setDefaults([
        'data_class' => Product::class,
        'require_sku' => false,
    ]);
    $resolver->setAllowedTypes('require_sku', 'bool');
}
```

Les options custom sont lues dans `buildForm` via `$options['require_sku']`.

### 5 — Services injectés

```php
public function __construct(private readonly Security $security) {}

public function buildForm(FormBuilderInterface $builder, array $options): void
{
    $user = $this->security->getUser();
    // ... utiliser $user pour filtrer un query_builder, etc.
}
```

### 6 — Vérification

```bash
vendor/bin/phpstan analyse src/Form
```

Tester via `/symfony:form-handle` (controller) ou un `FormTestCase` dédié.

### 7 — Clôture

Afficher :

- FormType créé/modifié (chemin).
- Champs, types, options custom.
- Ce qui reste : contrôleur (→ `/symfony:form-handle`), template Twig (→ `/symfony:form-render`), transformer/events/collection si besoin (→ `/symfony:form-advanced`).

## Delta Sylius

- Form sur une resource Sylius surchargée → hériter de `AbstractResourceType` quand le vendor le fait, pas de `AbstractType`.
- Groupes de validation : `validation_groups: ['Default', 'sylius']` dans `configureOptions` pour que les contraintes Sylius passent.
- Data channel-scopée → injecter `ChannelContextInterface` et filtrer les `EntityType::query_builder` par channel. Sinon fuite entre boutiques.

## Argument optionnel

`/symfony:form-type ProductType Product` — génère `ProductType` mappé sur `Product`.

`/symfony:form-type src/Form/OrderType.php` — audit d'un type existant (types explicites, options, `data_class`).

`/symfony:form-type` sans argument — demande l'entité et le cas d'usage.

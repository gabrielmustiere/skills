---
name: email
description: Envoie, personnalise ou ajoute un e-mail Sylius via le service Sender ou un EmailManager (OrderEmailManager, ShipmentEmailManager), override de templates Twig par canal, e-mail custom via SyliusMailerBundle.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /email — E-mails Sylius

Tu aides à **envoyer, personnaliser ou ajouter un e-mail Sylius**. Sylius envoie plusieurs e-mails natifs (confirmation de compte, vérification, reset password, confirmation de commande, expédition, contact) via `SyliusMailerBundle`, et expose deux points d'entrée pour envoyer programmatiquement : le service **`Sender`** et les **EmailManager** dédiés.

Référence officielle : [docs.sylius.com/the-book/architecture/e-mails](https://docs.sylius.com/the-book/architecture/e-mails) · [SyliusMailerBundle](https://github.com/Sylius/SyliusMailerBundle/blob/master/docs/index.md) · [cookbook templates par canal](https://docs.sylius.com/the-cookbook/emails/how-to-customize-email-templates-per-channel).

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine.
2. Vérifier `sylius/sylius` (ou `sylius/mailer-bundle`) dans les dépendances.
   - Présent → OK.
   - Absent → *« Ce skill cible Sylius (SyliusMailerBundle). Je ne trouve pas `sylius/sylius`. On bascule sur `symfony/mailer` standalone ou on continue quand même ? »*
3. Si besoin d'envoyer depuis un message bus / handler, prévoir aussi `/symfony:messenger` en amont.

## Table des e-mails natifs

| Code | Template par défaut | Paramètres | Déclencheur |
|------|---------------------|------------|-------------|
| `user_registration` | `@SyliusShop/Email/userRegistration.html.twig` | `user`, `channel`, `localeCode` | Inscription client |
| `verification_token` | `@SyliusShop/Email/verification.html.twig` | `user`, `channel`, `localeCode` | Inscription client (lien de vérification) |
| `reset_password_token` | `@SyliusShop/Email/passwordReset.html.twig` | `user`, `channel`, `localeCode` | Demande de reset depuis le login |
| `order_confirmation` | `@SyliusShop/Email/orderConfirmation.html.twig` | `order`, `channel`, `localeCode` | Commande passée |
| `shipment_confirmation` | `@SyliusAdmin/Email/shipmentConfirmation.html.twig` | `shipment`, `order`, `channel`, `localeCode` | Démarrage de l'expédition |
| `contact_request` | `@SyliusShop/Email/contactRequest.html.twig` | `data`, `channel`, `localeCode` | Soumission du formulaire de contact |

Sylius Plus ajoute les e-mails de **return requests** (`sylius_plus_return_request_confirmation`, `*_accepted`, `*_rejected`, `*_resolution_changed`, `*_repaired_items_sent`) avec leur propre template sous `@SyliusPlusPlugin/Returns/Infrastructure/Resources/views/Emails/`.

Les codes sont disponibles en constantes :
- `Sylius\Bundle\CoreBundle\Mailer\Emails` (commande, expédition, contact)
- `Sylius\Bundle\UserBundle\Mailer\Emails` (registration, verification, password reset)

## Règles fondamentales

- **Préférer les constantes** aux chaînes magiques : `Emails::ORDER_CONFIRMATION` plutôt que `'order_confirmation'`.
- **Passer tous les paramètres documentés** (voir table ci-dessus). Un paramètre manquant fait planter le rendu Twig, pas l'envoi — l'erreur arrive tard.
- **Override par canal** : ne pas modifier les templates `@SyliusShop` ou `@SyliusAdmin` dans `vendor/`. Copier dans `templates/bundles/SyliusShopBundle/Email/` (override global) ou `templates/bundles/SyliusShopBundle/<ChannelCode>/Email/` (override par canal) — voir [cookbook](https://docs.sylius.com/the-cookbook/emails/how-to-customize-email-templates-per-channel).
- **Sender vs EmailManager** : utiliser `OrderEmailManager` / `ShipmentEmailManager` quand ils existent (logique métier pré-câblée : locale de l'order, canal, destinataire client). Utiliser `Sender` pour tout le reste ou un e-mail custom.
- **Ne jamais envoyer un e-mail depuis un controller en HTTP synchrone** pour les flows chargés (confirmation de commande, etc.) — passer par `messenger` et un handler asynchrone si le SMTP peut être lent (éviter de bloquer le checkout).
- **Destinataire** : `Sender::send()` attend un **tableau** de destinataires (`['foo@example.com']`), même pour un seul — c'est une erreur classique de passer une string.

## Déroulement

### Cas 1 — Envoyer un e-mail natif

**Via `Sender` (bas niveau, universel) :**

```php
use Sylius\Bundle\CoreBundle\Mailer\Emails;
use Sylius\Component\Mailer\Sender\SenderInterface;

final class NotifyCustomer
{
    public function __construct(
        private SenderInterface $sender,
    ) {}

    public function __invoke(OrderInterface $order): void
    {
        $this->sender->send(
            Emails::ORDER_CONFIRMATION,
            [$order->getCustomer()->getEmail()],
            [
                'order' => $order,
                'channel' => $order->getChannel(),
                'localeCode' => $order->getLocaleCode(),
            ],
        );
    }
}
```

Injection : le service s'appelle `sylius.email_sender` (alias de `SenderInterface`).

**Via `OrderEmailManager` (haut niveau, logique câblée) :**

```php
use Sylius\Bundle\ShopBundle\EmailManager\OrderEmailManagerInterface;

final class ConfirmOrderHandler
{
    public function __construct(
        private OrderEmailManagerInterface $orderEmailManager,
    ) {}

    public function __invoke(OrderInterface $order): void
    {
        $this->orderEmailManager->sendConfirmationEmail($order);
    }
}
```

Idem pour `ShipmentEmailManager::sendConfirmationEmail($shipment)` côté admin.

### Cas 2 — Customiser un template

1. **Identifier le template cible** dans la table ci-dessus (ex. `@SyliusShop/Email/orderConfirmation.html.twig`).
2. **Override global** : copier vers `templates/bundles/SyliusShopBundle/Email/orderConfirmation.html.twig`. Symfony résout l'override automatiquement.
3. **Override par canal** : créer `templates/bundles/SyliusShopBundle/<CHANNEL_CODE>/Email/orderConfirmation.html.twig`. Le channel code est celui de l'entité `Channel` (ex. `FASHION_WEB`, `B2B_US`).
4. **Variables Twig disponibles** : exactement celles de la table (pas besoin de les re-lister dans le template, elles sont dans le contexte).
5. **Tester** : dans Sylius dev, la `MailCatcher` / `Mailpit` locale affiche les e-mails rendus sans les expédier — voir `.env` pour `MAILER_DSN`.

### Cas 3 — Ajouter un e-mail custom

Trois fichiers à toucher.

**a) Déclarer l'e-mail dans `config/packages/sylius_mailer.yaml`** (ou `sylius_mailer` dans `_sylius.yaml`) :

```yaml
sylius_mailer:
    emails:
        supplier_notification:
            subject: 'Nouvelle commande fournisseur'
            template: 'emails/supplier_notification.html.twig'
```

Le `subject` peut être une traduction (`app.email.supplier_notification.subject` avec Symfony Translation).

**b) Créer le template** `templates/emails/supplier_notification.html.twig` :

```twig
{% extends '@SyliusShop/Email/layout.html.twig' %}

{% block subject %}{{ 'app.email.supplier_notification.subject'|trans }}{% endblock %}

{% block content %}
    <p>Bonjour {{ supplier.name }},</p>
    <p>Nouvelle commande {{ order.number }} pour vos produits.</p>
{% endblock %}
```

Hériter de `@SyliusShop/Email/layout.html.twig` pour récupérer le header/footer de marque.

**c) Envoyer l'e-mail** depuis un service :

```php
final class NotifySupplier
{
    public function __construct(
        private SenderInterface $sender,
    ) {}

    public function notify(SupplierInterface $supplier, OrderInterface $order): void
    {
        $this->sender->send(
            'supplier_notification',
            [$supplier->getEmail()],
            [
                'supplier' => $supplier,
                'order' => $order,
                'channel' => $order->getChannel(),
                'localeCode' => $order->getLocaleCode(),
            ],
        );
    }
}
```

**Bonnes pratiques :**
- Garder `channel` et `localeCode` dans les paramètres même pour un e-mail custom — les helpers Twig Sylius (`{{ 'label'|trans({}, 'messages', localeCode) }}`) en dépendent.
- Nommer le code en `snake_case` préfixé par un namespace applicatif (`app_supplier_notification`) si plusieurs modules envoient des e-mails, pour éviter les collisions avec les futurs e-mails natifs Sylius.

### Cas 4 — Envoi asynchrone

Si le checkout timeout à cause du SMTP, déplacer l'envoi dans un handler Messenger :

1. Créer un message `SendOrderConfirmation(int $orderId)`.
2. Dispatcher depuis le state machine callback à la place de `sendConfirmationEmail($order)`.
3. Handler qui recharge l'order et appelle `OrderEmailManager`.
4. Router la queue `async` dans `config/packages/messenger.yaml`.

Pour les détails du pattern Messenger → `/symfony:messenger`.

## Pièges fréquents

- **E-mail non envoyé en prod, OK en dev** : vérifier `MAILER_DSN` (pas de `smtp://localhost` en prod), et que `SyliusMailerBundle` n'est pas en `delivery_disabled: true` via une config d'env (historique Sylius 1.x).
- **Template introuvable** : le path du template dans `sylius_mailer.emails.<code>.template` est **relatif à `templates/`** — pas d'alias `@App` nécessaire pour un template projet.
- **Override ignoré** : vérifier le chemin `templates/bundles/Sylius<Bundle>Bundle/...` — le nom du bundle (Shop/Admin) doit matcher celui du template natif. Override dans `SyliusCoreBundle` si le template vient de Core.
- **Destinataire vide** : `$order->getCustomer()->getEmail()` est `null` pour les commandes guest si pas encore renseigné — checker avant d'appeler `send()`.
- **Locale fausse dans l'e-mail** : passer `$order->getLocaleCode()`, pas la locale courante de la requête admin. Un admin `fr_FR` qui valide une expédition d'une commande `en_US` doit envoyer l'e-mail en `en_US`.
- **Paramètre Twig manquant** : l'exception `Twig\Error\RuntimeError: Variable "channel" does not exist` survient au rendu, pas à l'envoi — tester le template en dev avec un vrai order.
- **Sylius Plus return request e-mails** : les templates sont dans `@SyliusPlusPlugin`, pas `@SyliusShop` — override dans `templates/bundles/SyliusPlusPlugin/Returns/Infrastructure/Resources/views/Emails/`.

## Clôture

Afficher :
- Code e-mail utilisé (ou créé) et constante associée si native.
- Fichiers touchés (template, `sylius_mailer.yaml`, service d'envoi).
- Destinataires et paramètres passés.
- Ce qui reste : traductions des sujets/contenus, test en local avec Mailpit, éventuel passage en async via Messenger.

## Argument optionnel

`/sylius:email order_confirmation` — cadre l'envoi ou l'override de l'e-mail `order_confirmation`.

`/sylius:email supplier_notification` — démarre la création d'un e-mail custom `supplier_notification`.

`/sylius:email` sans argument — demande le code e-mail et le cas d'usage (envoi, override, création).

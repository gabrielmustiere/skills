# Skills Claude Code — Symfony · Sylius · éditorial

Collection de skills Claude Code regroupées par **plugins thématiques**, installables dans n'importe quel projet via `/plugin`. Des outils que je façonne pour Claude Code et que je partage en libre-service.

- **Marketplace** : `gabrielmustiere`
- **Source** : `gabrielmustiere/skills` (ce repo)

## Installation

Dans une session Claude Code ouverte sur n'importe quel projet :

```
/plugin marketplace add gabrielmustiere/skills
/plugin install sylius@gabrielmustiere
/plugin install symfony@gabrielmustiere
/plugin install editorial@gabrielmustiere
/reload-plugins
```

Les skills d'un plugin sont toujours namespacées par le nom du plugin :

```
/symfony:doctrine-entity
/sylius:state-machine
/editorial:article-plan
```

Mettre à jour le catalogue : `/plugin marketplace update gabrielmustiere` puis `/reload-plugins`.

## Plugins disponibles

| Plugin | Version | Inventaire | Description |
| --- | --- | --- | --- |
| `sylius` | `0.27.1` | [17 skills](documentation/sylius.md) | Skills pour travailler avec Sylius (conventions, entités traduisibles, customization de modèle/form/grid/template/styles/dynamic/validation/state-machine/translation/fixtures, commandes, e-mails, promotions panier, coupons, ajustements). |
| `symfony` | `0.13.1` | [25 skills](documentation/symfony.md) | Skills Symfony/Doctrine groupées par domaine : **doctrine** (entity, migration, query), **events** (dispatch, listen, subscribe), **forms** (type, handle, render, advanced), **http** (controller-action, routing-define), **http-client** (request, response, async, test), **messenger** (async), **serializer** (use), **object-mapper**, **services** (define, wire, tags), **validation** (constraints, groups, use). |
| `editorial` | `0.4.1` | [3 skills](documentation/editorial.md) | Pipeline éditorial en trois étapes — `article-plan` (cadrage), `article` (rédaction guidée + vérifications + traduction) et `article-rework` (retouche chirurgicale d'une portion d'un article publié) — pour articles de blog et fiches side-project. Stack-agnostique : détecte Astro Content Collections, Next.js MDX, Hugo, Jekyll ou markdown brut. Artifacts unifiés sous `docs/story/a-NNN-slug/`. |

## Licence

Distribué sous licence [Apache 2.0](LICENSE). © 2026 Gabriel Mustiere.

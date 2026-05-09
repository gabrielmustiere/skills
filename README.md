![Forge](documentation/banner.png)

Collection de skills Claude Code regroupées par **plugins thématiques**, installables dans n'importe quel projet via `/plugin`. Une petite forge où je façonne mes outils pour Claude Code et que je partage en libre-service.

- **Marketplace** : `gabrielmustiere`
- **Source** : `gabrielmustiere/skills` (ce repo)

## Premier pas avec `workflow`

Le plugin `workflow` est le plus structurant de la Forge : il pilote tout le cycle de développement en trois tracks symétriques, stack-agnostique. Le geste de base :

1. **Phase 0 — `/workflow:vision`** (une fois par projet) → atelier de cadrage qui produit `docs/vision.md` (problème, audience, North Star, principes, anti-objectifs). Lu ensuite par `feature-pitch` pour challenger l'alignement de chaque feature.
2. **Choisir une track** selon la nature du besoin :
   - **Feature** (user-facing) : `/workflow:feature-pitch` → `/workflow:feature-design` → `/workflow:feature`
   - **Refacto** : `/workflow:refactor-plan` → `/workflow:refactor`
   - **Évolution technique** (perf, résilience, sécu, observabilité) : `/workflow:tech-plan` → `/workflow:tech`
3. **Fin de cycle, communes aux trois tracks** : `/workflow:review` → `/workflow:commit` → `/workflow:report` → `/workflow:sync`.

Chaque étape produit un artefact sous `docs/story/NNN-<f|r|t>-<slug>/` qui sert d'entrée à la suivante (`feature.md`, `design.md`, `plan.md`, `review.md`, `report.md`). Si tu hésites sur le skill à appeler, `/workflow:help` rappelle l'état du pipeline et le prochain geste possible.

## Installation

Dans une session Claude Code ouverte sur n'importe quel projet :

```
/plugin marketplace add gabrielmustiere/skills
/plugin install workflow@gabrielmustiere
/plugin install sylius@gabrielmustiere
/plugin install symfony@gabrielmustiere
/plugin install editorial@gabrielmustiere
/reload-plugins
```

Les skills d'un plugin sont toujours namespacées par le nom du plugin :

```
/workflow:help
/workflow:feature-pitch
/workflow:doc-feature
/symfony:doctrine-entity
/editorial:article-plan
```

Mettre à jour le catalogue : `/plugin marketplace update gabrielmustiere` puis `/reload-plugins`.

## Plugins disponibles

| Plugin | Version | Description |
| --- | --- | --- |
| `workflow` | `0.13.0` | Pipeline de développement stack-agnostique. **Phase 0** : `vision` produit `docs/vision.md` (problème, audience, North Star, principes, anti-objectifs) — document fondateur lu par `feature-pitch` pour challenger l'alignement de chaque feature. Trois tracks symétriques : **feature** (`feature-pitch` → `feature-design` → `feature`), **refacto** (`refactor-plan` → `refactor`), **évolution technique** (`tech-plan` → `tech`). Étapes communes : `review` → `commit` → `report` → `sync`. Outillage transverse : `migrate-legacy`, `import-external`, `release`, `doc-feature` (carte d'une feature existante, stack-aware). Détection auto du stack (Symfony, Sylius). |
| `sylius` | `0.26.0` | Skills pour travailler avec Sylius (conventions, entités traduisibles, customization de modèle/form/grid/template/styles/dynamic/validation/state-machine/translation/fixtures, commandes, e-mails, promotions panier, coupons, ajustements). Pour la documentation d'une feature existante, voir `workflow:doc-feature`. |
| `symfony` | `0.12.0` | 25 skills Symfony/Doctrine groupées par domaine : **doctrine** (entity, migration, query), **events** (dispatch, listen, subscribe), **forms** (type, handle, render, advanced), **http** (controller-action, routing-define), **http-client** (request, response, async, test), **messenger** (async), **serializer** (use), **object-mapper**, **services** (define, wire, tags), **validation** (constraints, groups, use). Relayées par `workflow` quand le stack détecté est Symfony/Sylius. |
| `editorial` | `0.2.0` | Pipeline éditorial en trois étapes — `article-plan` (cadrage), `article` (rédaction guidée + vérifications + traduction) et `article-rework` (retouche chirurgicale d'une portion d'un article publié) — pour articles de blog et fiches side-project. Stack-agnostique : détecte Astro Content Collections, Next.js MDX, Hugo, Jekyll ou markdown brut. Artifacts unifiés sous `docs/story/a-NNN-slug/`. |

## Inventaire des skills

### `workflow` — Pipeline de développement (18 skills)

| Skill | Rôle |
| --- | --- |
| [`help`](plugins/workflow/skills/help/SKILL.md) | Sommaire du workflow, tracks, skills et artifacts |
| [`vision`](plugins/workflow/skills/vision/SKILL.md) | **Phase 0** — atelier de cadrage de la vision projet (problème, audience, valeur, North Star, principes, anti-objectifs) → `docs/vision.md` |
| [`feature-pitch`](plugins/workflow/skills/feature-pitch/SKILL.md) | Atelier de cadrage d'une idée de feature → `feature.md` |
| [`feature-design`](plugins/workflow/skills/feature-design/SKILL.md) | Design technique d'une feature cadrée → `design.md` |
| [`feature`](plugins/workflow/skills/feature/SKILL.md) | Implémentation guidée à partir du design |
| [`refactor-plan`](plugins/workflow/skills/refactor-plan/SKILL.md) | Cadrage refacto + tests de caractérisation → `plan.md` |
| [`refactor`](plugins/workflow/skills/refactor/SKILL.md) | Exécution guidée d'un refacto avec verrou tests |
| [`tech-plan`](plugins/workflow/skills/tech-plan/SKILL.md) | Cadrage évolution technique (perf, résilience, sécu) → `plan.md` |
| [`tech`](plugins/workflow/skills/tech/SKILL.md) | Exécution d'une évolution technique avec baseline/kill switch |
| [`review`](plugins/workflow/skills/review/SKILL.md) | Code review du diff (sécurité, qualité, conformité) |
| [`commit`](plugins/workflow/skills/commit/SKILL.md) | Génère un Conventional Commit FR et push |
| [`report`](plugins/workflow/skills/report/SKILL.md) | Compte rendu intention vs code réel |
| [`sync`](plugins/workflow/skills/sync/SKILL.md) | Réaligne la doc d'intention avec le code livré |
| [`test-scenario`](plugins/workflow/skills/test-scenario/SKILL.md) | Joue un scénario utilisateur via Playwright MCP |
| [`migrate-legacy`](plugins/workflow/skills/migrate-legacy/SKILL.md) | Migre les anciens dossiers `<f\|r\|t>-NNN-<slug>/` vers `NNN-<f\|r\|t>-<slug>/` (compteur en tête) via `git mv` |
| [`import-external`](plugins/workflow/skills/import-external/SKILL.md) | Importe une doc Spec Kit / BMAD-METHOD / GSD vers le format `docs/story/NNN-<f\|r\|t>-<slug>/` |
| [`release`](plugins/workflow/skills/release/SKILL.md) | Tag annoté SemVer + `CHANGELOG.md` Keep a Changelog + release GitHub |
| [`doc-feature`](plugins/workflow/skills/doc-feature/SKILL.md) | Documente une feature existante (stack-agnostique, détection Sylius/Symfony) → `docs/feature-map/NNN-slug/feature.md` |

### `sylius` — Skills Sylius (17 skills)

| Skill | Rôle |
| --- | --- |
| [`model`](plugins/sylius/skills/model/SKILL.md) | Étend un modèle Sylius natif (`Country`, `Customer`, `ShippingMethod`…) — extends + `implements Interface`, config `sylius_<bundle>.resources…classes.model`, migration |
| [`form`](plugins/sylius/skills/form/SKILL.md) | Étend un `FormType` Sylius via `AbstractTypeExtension` (shop vs admin vs base), priorité sur forms déjà étendus par le Core, Twig Hook pour le rendu, champs dynamiques via `PRE_SET_DATA` |
| [`grid`](plugins/sylius/skills/grid/SKILL.md) | Customise une grid Sylius (liste admin) — YAML dans `_sylius.yaml` (disable/réordonner champs, filtres, actions), PHP event listener `sylius.grid.<name>` via `GridDefinitionConverterEvent`, nouvelle grid sur resource custom via `AbstractGrid` + `#[AsGrid]`, filtres sur relations via `setRepositoryMethod()` ou `DataProviderInterface`, tri, pagination |
| [`template`](plugins/sylius/skills/template/SKILL.md) | Customise un template Sylius via Twig Hooks (recommandé), override `templates/bundles/` en dernier recours, ou Sylius Themes — hooks hiérarchiques, priorités par pas de 100, `enabled: false`, Twig Components |
| [`menu`](plugins/sylius/skills/menu/SKILL.md) | Customise un menu Sylius 2.x — sidebar admin/compte client via Event Listener (`MenuBuilderEvent`, events `sylius.menu.admin.main` / `sylius.menu.shop.account`), boutons d'action et onglets form produit/variante via Twig Hooks, icônes Tabler via `ux_icon()`, migration des events 1.x dépréciés |
| [`styles`](plugins/sylius/skills/styles/SKILL.md) | Customise les styles Sylius 2.x (admin Tabler / shop Bootstrap) — override des variables `--tblr-*` / `--bs-*` dans `assets/<contexte>/styles/custom.scss`, import entrypoint, `yarn build`, `cache:clear` + hard refresh |
| [`dynamic`](plugins/sylius/skills/dynamic/SKILL.md) | Customise les éléments dynamiques Sylius 2.1+ (Symfony UX + Stimulus) — auto-discovery d'un `*_controller.js`, binding via `stimulus_controller()`, injection Twig Hook, `controllers.json` pour désactiver/gouverner un natif (`enabled: false`, `fetch: lazy\|eager`), enregistrement manuel via `bootstrap.js` |
| [`validation`](plugins/sylius/skills/validation/SKILL.md) | Customise la validation d'une resource Sylius via `config/validator/<Model>.yaml` + groupe custom rebranché sur `sylius.form.type.*.validation_groups` — cas standard (ex. min length `ProductTranslation.name`), cas spéciaux `ShippingMethodRule`/`PromotionRule`/`PromotionAction` via `ChannelCodeCollection` + paramètre indexé par rule key |
| [`state-machine`](plugins/sylius/skills/state-machine/SKILL.md) | Customise une state machine Sylius (Symfony Workflow) — ajout de `place`/`transition` via `config/packages/_sylius.yaml` + constantes `OrderCheckoutTransitions::GRAPH`, suppression/extension via `CompilerPassInterface` sur `state_machine.<graph>.definition` enregistré dans `Kernel::build()`, listeners sur `workflow.<graph>.completed.<transition>`, override d'un listener Sylius (`sylius.listener.workflow.*`), cas legacy Winzou |
| [`translation`](plugins/sylius/skills/translation/SKILL.md) | Customise les libellés traduits d'un resource Sylius — override via `translations/<domaine>.<locale>.yaml` (`messages`, `validators`, `flashes`, `security`), repérage clé via Symfony Profiler, priorité app > plugin > vendor, fallback locale, `cache:clear` obligatoire |
| [`fixtures`](plugins/sylius/skills/fixtures/SKILL.md) | Customise les fixtures Sylius — modification de la suite `default` dans `config/packages/sylius_fixtures.yaml` (currencies, channels, shipping/payment methods custom), extension d'un `ExampleFactory` + `Fixture` pour exposer un champ custom sur entité étendue (Factory + `configureResourceNode` + service taggé `sylius_fixtures.fixture`), listing via `sylius:fixtures:list`, chargement via `sylius:fixtures:load` |
| [`translation-entity`](plugins/sylius/skills/translation-entity/SKILL.md) | Crée la paire entité traduisible + `*Translation` (personal translations, `TranslatableTrait`, fallback locale) |
| [`order`](plugins/sylius/skills/order/SKILL.md) | Crée/manipule/fait transiter un `Order` (factories, `OrderItemQuantityModifier`, `OrderProcessor`, state machines) |
| [`email`](plugins/sylius/skills/email/SKILL.md) | Envoie ou personnalise un e-mail Sylius (`Sender`, `EmailManager`, templates par canal, `SyliusMailerBundle`) |
| [`cart-promotion`](plugins/sylius/skills/cart-promotion/SKILL.md) | Crée et applique une promotion panier (rules, actions, filtres taxon, priorité, exclusivité, `PromotionProcessor`) |
| [`coupon`](plugins/sylius/skills/coupon/SKILL.md) | Crée, applique ou génère en bulk des `PromotionCoupon` (promo `couponBased=true`, expirationDate, usageLimit, `PromotionCouponGenerator`) |
| [`adjustment`](plugins/sylius/skills/adjustment/SKILL.md) | Crée ou manipule un `Adjustment` sur `Order`/`OrderItem`/`OrderItemUnit` (types promo/shipping/tax, neutral, `lock()` survie aux recalculs) |

### `symfony` — Skills Symfony/Doctrine (25 skills)

| Domaine | Skill | Rôle |
| --- | --- | --- |
| **doctrine** | [`doctrine-entity`](plugins/symfony/skills/doctrine-entity/SKILL.md) | Entité Doctrine : ORM, relations, types custom |
|  | [`doctrine-query`](plugins/symfony/skills/doctrine-query/SKILL.md) | Requête Doctrine : find/findBy, DQL, QueryBuilder, DBAL |
|  | [`doctrine-migration`](plugins/symfony/skills/doctrine-migration/SKILL.md) | Migration : génération, dry-run, réversibilité |
| **events** | [`event-dispatch`](plugins/symfony/skills/event-dispatch/SKILL.md) | Définit et dispatche un événement custom |
|  | [`event-listen`](plugins/symfony/skills/event-listen/SKILL.md) | Event Listener (`#[AsEventListener]`) |
|  | [`event-subscribe`](plugins/symfony/skills/event-subscribe/SKILL.md) | Event Subscriber (multi-callbacks, priorités) |
| **forms** | [`form-type`](plugins/symfony/skills/form-type/SKILL.md) | Classe FormType (AbstractType, buildForm) |
|  | [`form-handle`](plugins/symfony/skills/form-handle/SKILL.md) | Traitement en contrôleur (createForm, handleRequest, PRG) |
|  | [`form-render`](plugins/symfony/skills/form-render/SKILL.md) | Rendu Twig (thèmes, customisation par bloc) |
|  | [`form-advanced`](plugins/symfony/skills/form-advanced/SKILL.md) | DataTransformer, FormEvents, CollectionType, FileType |
| **http** | [`controller-action`](plugins/symfony/skills/controller-action/SKILL.md) | Contrôleur fin (AbstractController, helpers, `#[MapQueryParameter]`, `#[MapRequestPayload]`) |
|  | [`routing-define`](plugins/symfony/skills/routing-define/SKILL.md) | `#[Route]` (path, methods, requirements, host, locale), génération d'URL, URIs signées |
| **http-client** | [`http-client-request`](plugins/symfony/skills/http-client-request/SKILL.md) | Construction d'une requête HTTP sortante |
|  | [`http-client-response`](plugins/symfony/skills/http-client-response/SKILL.md) | Consommation de réponse, exceptions, streaming |
|  | [`http-client-async`](plugins/symfony/skills/http-client-async/SKILL.md) | Concurrence, retry, cache, rate-limit, SSE |
|  | [`http-client-test`](plugins/symfony/skills/http-client-test/SKILL.md) | Mock HttpClient, MockResponse, HAR |
| **messenger** | [`messenger-async`](plugins/symfony/skills/messenger-async/SKILL.md) | Message + handler `#[AsMessageHandler]`, transports (doctrine/amqp/redis), retry, DLQ, worker |
| **serializer** | [`serializer-use`](plugins/symfony/skills/serializer-use/SKILL.md) | `SerializerInterface`, normalizers/encoders (json/xml/csv), `#[Groups]`, `#[Context]`, `#[DiscriminatorMap]` |
| **object-mapper** | [`object-mapper`](plugins/symfony/skills/object-mapper/SKILL.md) | Symfony ObjectMapper 7.3+ : `#[Map]` (target/source/if/transform), conditions, transformers, MapCollection |
| **services** | [`service-define`](plugins/symfony/skills/service-define/SKILL.md) | Déclaration (autowiring, autoconfiguration) |
|  | [`service-wire`](plugins/symfony/skills/service-wire/SKILL.md) | Câblage d'arguments (scalaires, env, alias, `#[Target]`) |
|  | [`service-tags`](plugins/symfony/skills/service-tags/SKILL.md) | Tags, collections, factories, décorateurs |
| **validation** | [`validation-constraints`](plugins/symfony/skills/validation-constraints/SKILL.md) | Contraintes `#[Assert\*]` sur entité/DTO/classe |
|  | [`validation-groups`](plugins/symfony/skills/validation-groups/SKILL.md) | Groupes, séquences, validation conditionnelle |
|  | [`validation-use`](plugins/symfony/skills/validation-use/SKILL.md) | `ValidatorInterface` hors form (DTO, payload, CLI) |

### `editorial` — Pipeline de rédaction (3 skills)

| Skill | Rôle |
| --- | --- |
| [`article-plan`](plugins/editorial/skills/article-plan/SKILL.md) | Atelier de cadrage d'un article de blog ou d'une fiche side-project — sujet, thèse, audience, recherche, chapitrage, tonalité, frontmatter prévisionnel adapté à la stack détectée → `docs/story/a-<NNN>-<slug>/plan.md` |
| [`article`](plugins/editorial/skills/article/SKILL.md) | Rédaction guidée à partir du `plan.md` validé — produit le fichier final dans la collection détectée (Astro CC, Next MDX, Hugo, Jekyll, markdown brut), frontmatter conforme au schéma, vérifications schéma + lint + format, traduction multilingue si prévue |
| [`article-rework`](plugins/editorial/skills/article-rework/SKILL.md) | Retouche chirurgicale d'une portion d'un article publié (chapitre, section, paragraphe) — respecte la voix de l'article, lit le `plan.md` associé, met à jour le plan si la promesse de la section change, propage à la traduction, vérifie schéma + lint + format |

# Skills

Collection de skills Claude Code. Les skills sont regroupées par **plugins thématiques** et installables dans n'importe quel projet via la commande `/plugin`.

- **Nom de la marketplace** : `gabrielmustiere`
- **Source** : `gabrielmustiere/skills` (ce repo)

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

Les skills d'un plugin sont toujours namespacées par le nom du plugin. Exemples d'invocation :

```
/workflow:help
/workflow:feature-pitch
/sylius:doc-sylius
/symfony:doctrine-entity
/editorial:article-plan
```

Mettre à jour quand le catalogue change : `/plugin marketplace update gabrielmustiere` puis `/reload-plugins`.

### Tester localement sans publier

Depuis n'importe quel projet :

```
claude --plugin-dir /Users/gabriel/projets/skills/plugins/workflow
```

Après modification d'une skill : `/reload-plugins` dans la session en cours (pas besoin de redémarrer). Plusieurs plugins en même temps : répéter `--plugin-dir`.

## Plugins disponibles

| Plugin | Version | Description |
| --- | --- | --- |
| `workflow` | `0.7.0` | Pipeline de développement stack-agnostique. Trois tracks symétriques : **feature** (`feature-pitch` → `feature-design` → `feature`), **refacto** (`refactor-plan` → `refactor`), **évolution technique** (`tech-plan` → `tech`). Étapes communes : `review` → `commit` → `report` → `sync`. Détection auto du stack (Symfony, Sylius). |
| `sylius` | `0.24.0` | Skills pour travailler avec Sylius (doc, conventions, entités traduisibles, customization de modèle/form/grid/template/styles/dynamic/validation/state-machine/translation/fixtures, commandes, e-mails, promotions panier, coupons, ajustements). |
| `symfony` | `0.12.0` | 25 skills Symfony/Doctrine groupées par domaine : **doctrine** (entity, migration, query), **events** (dispatch, listen, subscribe), **forms** (type, handle, render, advanced), **http** (controller-action, routing-define), **http-client** (request, response, async, test), **messenger** (async), **serializer** (use), **object-mapper**, **services** (define, wire, tags), **validation** (constraints, groups, use). Relayées par `workflow` quand le stack détecté est Symfony/Sylius. |
| `editorial` | `0.1.0` | Pipeline éditorial en deux étapes — `article-plan` (cadrage) puis `article` (rédaction guidée + vérifications + traduction) — pour articles de blog et fiches side-project. Stack-agnostique : détecte Astro Content Collections, Next.js MDX, Hugo, Jekyll ou markdown brut. Artifacts unifiés sous `docs/story/a-NNN-slug/`. |

## Inventaire des skills

### `workflow` — Pipeline de développement (13 skills)

| Skill | Rôle |
| --- | --- |
| [`help`](plugins/workflow/skills/help/SKILL.md) | Sommaire du workflow, tracks, skills et artifacts |
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

### `sylius` — Skills Sylius (18 skills)

| Skill | Rôle |
| --- | --- |
| [`doc-sylius`](plugins/sylius/skills/doc-sylius/SKILL.md) | Documente une feature Sylius (custom ou vendor) → `docs/sylius-native/` |
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

### `editorial` — Pipeline de rédaction (2 skills)

| Skill | Rôle |
| --- | --- |
| [`article-plan`](plugins/editorial/skills/article-plan/SKILL.md) | Atelier de cadrage d'un article de blog ou d'une fiche side-project — sujet, thèse, audience, recherche, chapitrage, tonalité, frontmatter prévisionnel adapté à la stack détectée → `docs/story/a-<NNN>-<slug>/plan.md` |
| [`article`](plugins/editorial/skills/article/SKILL.md) | Rédaction guidée à partir du `plan.md` validé — produit le fichier final dans la collection détectée (Astro CC, Next MDX, Hugo, Jekyll, markdown brut), frontmatter conforme au schéma, vérifications schéma + lint + format, traduction multilingue si prévue |

## Structure du repo

```
.
├── .claude-plugin/
│   └── marketplace.json         ← catalogue (liste les plugins)
└── plugins/
    └── workflow/                 ← un plugin thématique
        ├── .claude-plugin/
        │   └── plugin.json       ← manifeste du plugin
        └── skills/
            └── help/
                └── SKILL.md      ← une skill
```

Règle : `skills/`, `commands/`, `agents/`, `hooks/` sont à la **racine du plugin**, jamais dans `.claude-plugin/`.

## Ajouter une skill

### Dans un plugin thématique existant

1. Créer `plugins/<plugin>/skills/<nom-skill>/SKILL.md` avec un frontmatter :
   ```yaml
   ---
   name: nom-skill
   description: Ce que fait la skill et quand l'utiliser.
   ---
   ```
2. Bumper la `version` dans `plugins/<plugin>/.claude-plugin/plugin.json` (semver)
3. `git push`

Aucune modif de `marketplace.json` nécessaire — la skill est auto-découverte dans le plugin.

### Nouveau plugin thématique

1. Créer `plugins/<nouveau>/.claude-plugin/plugin.json` (copier celui de `workflow` et adapter `name`/`description`)
2. Créer au moins une skill dans `plugins/<nouveau>/skills/<skill>/SKILL.md`
3. Ajouter une entrée dans `.claude-plugin/marketplace.json` :
   ```json
   {
     "name": "<nouveau>",
     "source": "./plugins/<nouveau>",
     "description": "...",
     "version": "0.1.0"
   }
   ```
4. `git push`

Côté utilisateurs : `/plugin marketplace update gabrielmustiere` puis `/plugin install <nouveau>@gabrielmustiere`.

## Versionnage

Deux niveaux de versions, **indépendants** :

- **Marketplace (ce repo)** — un schéma semver unifié `vX.Y.Z` matérialisé par les **tags Git et les GitHub Releases**. Monotone, une seule séquence pour toutes les releases. C'est ce que l'utilisateur voit dans la liste des releases GitHub.
- **Plugin** — chaque `plugin.json` a sa propre `version` (semver). Sert à déclencher la mise à jour côté utilisateurs et à indiquer la maturité interne du plugin. **N'est pas reflétée dans les tags Git.**

### Bumper la marketplace

Règles pour le tag global après un `git push` :
- **Patch** (`v0.5.0` → `v0.5.1`) : fix isolé, doc, refacto interne.
- **Minor** (`v0.5.0` → `v0.6.0`) : nouveau plugin, nouvelles skills, ajouts non breaking.
- **Major** (`v0.x` → `v1.0.0`) : restructuration de la marketplace, breaking côté utilisateurs.

```
git tag -a v0.6.0 -m "v0.6.0 — <résumé>"
git push origin v0.6.0
gh release create v0.6.0 --title "v0.6.0 — <résumé>" --notes "..."
```

### Bumper un plugin

Indépendant du tag global. Bumper la `version` dans `plugin.json` selon ses propres changements (semver classique). Ce numéro est ce que voit l'utilisateur quand il fait `/plugin install <plugin>@gabrielmustiere`.

## Référence frontmatter SKILL.md

Champs optionnels utiles (voir [doc officielle](https://code.claude.com/docs/fr/skills#frontmatter-reference)) :

| Champ                      | Usage                                                                          |
| -------------------------- | ------------------------------------------------------------------------------ |
| `description`              | Recommandé. Aide Claude à décider quand charger la skill automatiquement.      |
| `disable-model-invocation` | `true` = seul l'utilisateur peut invoquer (pour `/deploy`, `/commit`, etc.).   |
| `allowed-tools`            | Outils autorisés sans demander permission quand la skill est active.           |
| `paths`                    | Globs qui limitent l'auto-activation à certains fichiers.                      |
| `argument-hint`            | Hint d'autocomplétion, ex : `[issue-number]`.                                  |

Substitutions disponibles dans le contenu : `$ARGUMENTS`, `$0`, `$1`, `${CLAUDE_SKILL_DIR}`, `${CLAUDE_SESSION_ID}`.

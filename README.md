# Skills

Marketplace personnelle de skills Claude Code. Les skills sont regroupées par **plugins thématiques** et installables dans n'importe quel projet via la commande `/plugin`.

- **Nom de la marketplace** : `gabrielmustiere`
- **Source** : `gabrielmustiere/skills` (ce repo)

## Plugins disponibles

| Plugin | Version | Description |
| --- | --- | --- |
| `workflow` | `0.7.0` | Pipeline de développement stack-agnostique. Trois tracks symétriques : **feature** (`feature-pitch` → `feature-design` → `feature`), **refacto** (`refactor-plan` → `refactor`), **évolution technique** (`tech-plan` → `tech`). Étapes communes : `review` → `commit` → `report` → `sync`. Détection auto du stack (Symfony, Sylius). |
| `sylius` | `0.24.0` | Skills pour travailler avec Sylius (doc, conventions, entités traduisibles, customization de modèle/form/grid/template/styles/dynamic/validation/state-machine/translation/fixtures, commandes, e-mails, promotions panier, coupons, ajustements). |
| `symfony` | `0.12.0` | 25 skills Symfony/Doctrine groupées par domaine : **doctrine** (entity, migration, query), **events** (dispatch, listen, subscribe), **forms** (type, handle, render, advanced), **http** (controller-action, routing-define), **http-client** (request, response, async, test), **messenger** (async), **serializer** (use), **object-mapper**, **services** (define, wire, tags), **validation** (constraints, groups, use). Relayées par `workflow` quand le stack détecté est Symfony/Sylius. |

## Inventaire des skills

### `workflow` — Pipeline de développement (13 skills)

| Skill | Rôle |
| --- | --- |
| `help` | Sommaire du workflow, tracks, skills et artifacts |
| `feature-pitch` | Atelier de cadrage d'une idée de feature → `feature.md` |
| `feature-design` | Design technique d'une feature cadrée → `design.md` |
| `feature` | Implémentation guidée à partir du design |
| `refactor-plan` | Cadrage refacto + tests de caractérisation → `plan.md` |
| `refactor` | Exécution guidée d'un refacto avec verrou tests |
| `tech-plan` | Cadrage évolution technique (perf, résilience, sécu) → `plan.md` |
| `tech` | Exécution d'une évolution technique avec baseline/kill switch |
| `review` | Code review du diff (sécurité, qualité, conformité) |
| `commit` | Génère un Conventional Commit FR et push |
| `report` | Compte rendu intention vs code réel |
| `sync` | Réaligne la doc d'intention avec le code livré |
| `test-scenario` | Joue un scénario utilisateur via Playwright MCP |

### `sylius` — Skills Sylius (18 skills)

| Skill | Rôle |
| --- | --- |
| `doc-sylius` | Documente une feature Sylius (custom ou vendor) → `docs/sylius-native/` |
| `model` | Étend un modèle Sylius natif (`Country`, `Customer`, `ShippingMethod`…) — extends + `implements Interface`, config `sylius_<bundle>.resources…classes.model`, migration |
| `form` | Étend un `FormType` Sylius via `AbstractTypeExtension` (shop vs admin vs base), priorité sur forms déjà étendus par le Core, Twig Hook pour le rendu, champs dynamiques via `PRE_SET_DATA` |
| `grid` | Customise une grid Sylius (liste admin) — YAML dans `_sylius.yaml` (disable/réordonner champs, filtres, actions), PHP event listener `sylius.grid.<name>` via `GridDefinitionConverterEvent`, nouvelle grid sur resource custom via `AbstractGrid` + `#[AsGrid]`, filtres sur relations via `setRepositoryMethod()` ou `DataProviderInterface`, tri, pagination |
| `template` | Customise un template Sylius via Twig Hooks (recommandé), override `templates/bundles/` en dernier recours, ou Sylius Themes — hooks hiérarchiques, priorités par pas de 100, `enabled: false`, Twig Components |
| `menu` | Customise un menu Sylius 2.x — sidebar admin/compte client via Event Listener (`MenuBuilderEvent`, events `sylius.menu.admin.main` / `sylius.menu.shop.account`), boutons d'action et onglets form produit/variante via Twig Hooks, icônes Tabler via `ux_icon()`, migration des events 1.x dépréciés |
| `styles` | Customise les styles Sylius 2.x (admin Tabler / shop Bootstrap) — override des variables `--tblr-*` / `--bs-*` dans `assets/<contexte>/styles/custom.scss`, import entrypoint, `yarn build`, `cache:clear` + hard refresh |
| `dynamic` | Customise les éléments dynamiques Sylius 2.1+ (Symfony UX + Stimulus) — auto-discovery d'un `*_controller.js`, binding via `stimulus_controller()`, injection Twig Hook, `controllers.json` pour désactiver/gouverner un natif (`enabled: false`, `fetch: lazy|eager`), enregistrement manuel via `bootstrap.js` |
| `validation` | Customise la validation d'une resource Sylius via `config/validator/<Model>.yaml` + groupe custom rebranché sur `sylius.form.type.*.validation_groups` — cas standard (ex. min length `ProductTranslation.name`), cas spéciaux `ShippingMethodRule`/`PromotionRule`/`PromotionAction` via `ChannelCodeCollection` + paramètre indexé par rule key |
| `state-machine` | Customise une state machine Sylius (Symfony Workflow) — ajout de `place`/`transition` via `config/packages/_sylius.yaml` + constantes `OrderCheckoutTransitions::GRAPH`, suppression/extension via `CompilerPassInterface` sur `state_machine.<graph>.definition` enregistré dans `Kernel::build()`, listeners sur `workflow.<graph>.completed.<transition>`, override d'un listener Sylius (`sylius.listener.workflow.*`), cas legacy Winzou |
| `translation` | Customise les libellés traduits d'un resource Sylius — override via `translations/<domaine>.<locale>.yaml` (`messages`, `validators`, `flashes`, `security`), repérage clé via Symfony Profiler, priorité app > plugin > vendor, fallback locale, `cache:clear` obligatoire |
| `fixtures` | Customise les fixtures Sylius — modification de la suite `default` dans `config/packages/sylius_fixtures.yaml` (currencies, channels, shipping/payment methods custom), extension d'un `ExampleFactory` + `Fixture` pour exposer un champ custom sur entité étendue (Factory + `configureResourceNode` + service taggé `sylius_fixtures.fixture`), listing via `sylius:fixtures:list`, chargement via `sylius:fixtures:load` |
| `translation-entity` | Crée la paire entité traduisible + `*Translation` (personal translations, `TranslatableTrait`, fallback locale) |
| `order` | Crée/manipule/fait transiter un `Order` (factories, `OrderItemQuantityModifier`, `OrderProcessor`, state machines) |
| `email` | Envoie ou personnalise un e-mail Sylius (`Sender`, `EmailManager`, templates par canal, `SyliusMailerBundle`) |
| `cart-promotion` | Crée et applique une promotion panier (rules, actions, filtres taxon, priorité, exclusivité, `PromotionProcessor`) |
| `coupon` | Crée, applique ou génère en bulk des `PromotionCoupon` (promo `couponBased=true`, expirationDate, usageLimit, `PromotionCouponGenerator`) |
| `adjustment` | Crée ou manipule un `Adjustment` sur `Order`/`OrderItem`/`OrderItemUnit` (types promo/shipping/tax, neutral, `lock()` survie aux recalculs) |

### `symfony` — Skills Symfony/Doctrine (25 skills)

| Domaine | Skill | Rôle |
| --- | --- | --- |
| **doctrine** | `doctrine-entity` | Entité Doctrine : ORM, relations, types custom |
|  | `doctrine-query` | Requête Doctrine : find/findBy, DQL, QueryBuilder, DBAL |
|  | `doctrine-migration` | Migration : génération, dry-run, réversibilité |
| **events** | `event-dispatch` | Définit et dispatche un événement custom |
|  | `event-listen` | Event Listener (`#[AsEventListener]`) |
|  | `event-subscribe` | Event Subscriber (multi-callbacks, priorités) |
| **forms** | `form-type` | Classe FormType (AbstractType, buildForm) |
|  | `form-handle` | Traitement en contrôleur (createForm, handleRequest, PRG) |
|  | `form-render` | Rendu Twig (thèmes, customisation par bloc) |
|  | `form-advanced` | DataTransformer, FormEvents, CollectionType, FileType |
| **http** | `controller-action` | Contrôleur fin (AbstractController, helpers, `#[MapQueryParameter]`, `#[MapRequestPayload]`) |
|  | `routing-define` | `#[Route]` (path, methods, requirements, host, locale), génération d'URL, URIs signées |
| **http-client** | `http-client-request` | Construction d'une requête HTTP sortante |
|  | `http-client-response` | Consommation de réponse, exceptions, streaming |
|  | `http-client-async` | Concurrence, retry, cache, rate-limit, SSE |
|  | `http-client-test` | Mock HttpClient, MockResponse, HAR |
| **messenger** | `messenger-async` | Message + handler `#[AsMessageHandler]`, transports (doctrine/amqp/redis), retry, DLQ, worker |
| **serializer** | `serializer-use` | `SerializerInterface`, normalizers/encoders (json/xml/csv), `#[Groups]`, `#[Context]`, `#[DiscriminatorMap]` |
| **object-mapper** | `object-mapper` | Symfony ObjectMapper 7.3+ : `#[Map]` (target/source/if/transform), conditions, transformers, MapCollection |
| **services** | `service-define` | Déclaration (autowiring, autoconfiguration) |
|  | `service-wire` | Câblage d'arguments (scalaires, env, alias, `#[Target]`) |
|  | `service-tags` | Tags, collections, factories, décorateurs |
| **validation** | `validation-constraints` | Contraintes `#[Assert\*]` sur entité/DTO/classe |
|  | `validation-groups` | Groupes, séquences, validation conditionnelle |
|  | `validation-use` | `ValidatorInterface` hors form (DTO, payload, CLI) |

## Installer dans un autre projet

Dans une session Claude Code ouverte sur n'importe quel projet :

```
/plugin marketplace add gabrielmustiere/skills
/plugin install workflow@gabrielmustiere
/plugin install sylius@gabrielmustiere
/plugin install symfony@gabrielmustiere
/reload-plugins
```

Les skills d'un plugin sont toujours namespacées par le nom du plugin. Exemples d'invocation :

```
/workflow:help
/workflow:feature-pitch
/sylius:doc-sylius
/symfony:doctrine-entity
```

Mettre à jour quand le catalogue change : `/plugin marketplace update gabrielmustiere` puis `/reload-plugins`.

## Tester localement sans publier

Depuis n'importe quel projet :

```
claude --plugin-dir /Users/gabriel/projets/skills/plugins/workflow
```

Après modification d'une skill : `/reload-plugins` dans la session en cours (pas besoin de redémarrer). Plusieurs plugins en même temps : répéter `--plugin-dir`.

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
     "source": "<nouveau>",
     "description": "...",
     "version": "0.1.0"
   }
   ```
4. `git push`

Côté utilisateurs : `/plugin marketplace update gabrielmustiere` puis `/plugin install <nouveau>@gabrielmustiere`.

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

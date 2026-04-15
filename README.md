# Skills

Marketplace personnelle de skills Claude Code. Les skills sont regroupées par **plugins thématiques** et installables dans n'importe quel projet via la commande `/plugin`.

- **Nom de la marketplace** : `gabrielmustiere`
- **Source** : `gabrielmustiere/skills` (ce repo)

## Plugins disponibles

| Plugin | Version | Description |
| --- | --- | --- |
| `workflow` | `0.7.0` | Pipeline de développement stack-agnostique. Trois tracks symétriques : **feature** (`feature-pitch` → `feature-design` → `feature`), **refacto** (`refactor-plan` → `refactor`), **évolution technique** (`tech-plan` → `tech`). Étapes communes : `review` → `commit` → `report` → `sync`. Détection auto du stack (Symfony, Sylius). |
| `sylius` | `0.6.0` | Skills pour travailler avec Sylius (doc et conventions). |
| `symfony` | `0.7.0` | 20 skills Symfony/Doctrine groupées par domaine : **doctrine** (entity, migration, query), **events** (dispatch, listen, subscribe), **forms** (type, handle, render, advanced), **http-client** (request, response, async, test), **services** (define, wire, tags), **validation** (constraints, groups, use). Relayées par `workflow` quand le stack détecté est Symfony/Sylius. |

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

### `sylius` — Documentation Sylius (1 skill)

| Skill | Rôle |
| --- | --- |
| `doc-sylius` | Documente une feature Sylius (custom ou vendor) → `docs/sylius-native/` |

### `symfony` — Skills Symfony/Doctrine (20 skills)

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
| **http-client** | `http-client-request` | Construction d'une requête HTTP sortante |
|  | `http-client-response` | Consommation de réponse, exceptions, streaming |
|  | `http-client-async` | Concurrence, retry, cache, rate-limit, SSE |
|  | `http-client-test` | Mock HttpClient, MockResponse, HAR |
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

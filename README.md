# skills

Marketplace personnelle de skills Claude Code. Les skills sont regroupées par **plugins thématiques** et installables dans n'importe quel projet via la commande `/plugin`.

- **Nom de la marketplace** : `gabrielmustiere`
- **Source** : `gabrielmustiere/skills` (ce repo)

## Installer dans un autre projet

Dans une session Claude Code ouverte sur n'importe quel projet :

```
/plugin marketplace add gabrielmustiere/skills
/plugin install dev-workflow@gabrielmustiere
/reload-plugins
```

Les skills d'un plugin sont toujours namespacées par le nom du plugin. Exemple d'invocation :

```
/dev-workflow:sp-help
```

Mettre à jour quand le catalogue change : `/plugin marketplace update gabrielmustiere` puis `/reload-plugins`.

## Tester localement sans publier

Depuis n'importe quel projet :

```
claude --plugin-dir /Users/gabriel/projets/skills/plugins/dev-workflow
```

Après modification d'une skill : `/reload-plugins` dans la session en cours (pas besoin de redémarrer). Plusieurs plugins en même temps : répéter `--plugin-dir`.

## Structure du repo

```
.
├── .claude-plugin/
│   └── marketplace.json         ← catalogue (liste les plugins)
└── plugins/
    └── dev-workflow/             ← un plugin thématique
        ├── .claude-plugin/
        │   └── plugin.json       ← manifeste du plugin
        └── skills/
            └── sp-help/
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

1. Créer `plugins/<nouveau>/.claude-plugin/plugin.json` (copier celui de `dev-workflow` et adapter `name`/`description`)
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

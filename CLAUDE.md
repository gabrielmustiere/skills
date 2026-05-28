# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Nature du repo

Ce repo **n'est pas une application** : c'est une **marketplace de plugins Claude Code** (nom public : `gabrielmustiere`) distribuée via GitHub. Il n'y a ni build, ni test, ni runtime — juste du JSON et du Markdown consommés par Claude Code chez les utilisateurs qui ajoutent cette marketplace avec `/plugin marketplace add gabrielmustiere/skills`. Elle publie les plugins `sylius`, `symfony` et `editorial`.

Le plugin `workflow` (pipeline de développement) a été extrait dans un repo dédié : `gabrielmustiere/forge` (marketplace `forge`). Toute évolution de ce plugin se fait là-bas, pas ici.

Source de vérité :
- `.claude-plugin/marketplace.json` → catalogue listant les plugins publiés
- `plugins/<nom>/.claude-plugin/plugin.json` → manifeste d'un plugin
- `plugins/<nom>/skills/<skill>/SKILL.md` → une skill (frontmatter YAML + instructions Markdown)
- `documentation/<plugin>.md` → inventaire lisible (2 colonnes skill / rôle) d'un plugin, un fichier par plugin. À mettre à jour à chaque ajout/retrait/renommage de skill.

Références externes : [docs plugins](https://code.claude.com/docs/fr/plugins), [docs skills](https://code.claude.com/docs/fr/skills), [docs marketplaces](https://code.claude.com/docs/fr/plugin-marketplaces).

## Architecture

```
.claude-plugin/marketplace.json        ← catalogue (name, owner, pluginRoot, plugins[])
plugins/<plugin-name>/
  .claude-plugin/plugin.json           ← manifeste (name, description, version, author)
  skills/<skill-name>/SKILL.md         ← une skill, nom du dossier = nom de la skill
```

Règle structurelle critique : `skills/`, `commands/`, `agents/`, `hooks/` vont **à la racine du plugin**, jamais dans `.claude-plugin/`. Seul `plugin.json` habite `.claude-plugin/`.

Granularité : **plugins thématiques**. Un plugin regroupe plusieurs skills liées (ex: `symfony` contient toutes les skills Symfony/Doctrine groupées par domaine). Les utilisateurs installent un thème entier, pas skill par skill.

Namespacing : les skills de plugin sont toujours invoquées en préfixant par le nom du plugin → `/symfony:doctrine-entity`, pas `/doctrine-entity`. Le préfixe vient du champ `name` dans `plugin.json`.

Résolution des `source` dans `marketplace.json` : `metadata.pluginRoot: "./plugins"` permet d'écrire `"source": "sylius"` au lieu de `"source": "./plugins/sylius"`.

## Workflow d'édition

### Ajouter une skill à un plugin existant
1. Créer `plugins/<plugin>/skills/<nouveau-skill>/SKILL.md` avec frontmatter `name` + `description`
2. Bumper `version` dans `plugins/<plugin>/.claude-plugin/plugin.json` (semver)
3. Ajouter une ligne (skill / rôle) dans `documentation/<plugin>.md` et mettre à jour le compteur de skills du `README.md` (colonne « Inventaire »)
4. `git push`
5. Aucune modif de `marketplace.json` — les skills sont auto-découvertes dans le plugin

### Créer un nouveau plugin thématique
1. `plugins/<nouveau>/.claude-plugin/plugin.json` (copier `symfony/` comme base)
2. Au moins une skill dans `plugins/<nouveau>/skills/<skill>/SKILL.md`
3. Ajouter une entrée au tableau `plugins` de `.claude-plugin/marketplace.json`
4. Créer `documentation/<nouveau>.md` (inventaire 2 colonnes) et ajouter la ligne correspondante dans le tableau « Plugins disponibles » du `README.md`
5. `git push`

### Tester localement avant push
Depuis n'importe quel projet :
```
claude --plugin-dir /Users/gabriel/projets/skills/plugins/<plugin-name>
```
Pendant la session : `/reload-plugins` après chaque modif (pas besoin de redémarrer).

### Installation côté utilisateur
```
/plugin marketplace add gabrielmustiere/skills
/plugin install <plugin>@gabrielmustiere
/reload-plugins
```
Pull des maj : `/plugin marketplace update gabrielmustiere`.

## Conventions

- Kebab-case partout (noms de plugins, noms de skills, noms de dossiers) — le `name` YAML du SKILL.md doit matcher le nom du dossier
- Français dans les `description` (cohérent avec les skills existantes)
- Semver dans chaque `plugin.json`
- Frontmatter SKILL.md minimal = `name` + `description`. Champs utiles selon la skill :
  - `disable-model-invocation: true` → empêche Claude d'invoquer automatiquement (pour skills à effets de bord type `/commit`, `/deploy`)
  - `user_invocable: true` → force la visibilité dans le menu `/` même si autres flags la masqueraient
  - `allowed-tools` → outils autorisés sans demande de permission
  - `paths` → auto-activation limitée à certains globs de fichiers
- Substitutions dispo dans le contenu SKILL.md : `$ARGUMENTS`, `$0`, `$1`, `${CLAUDE_SKILL_DIR}`, `${CLAUDE_SESSION_ID}`

## Piège fréquent

Une skill qui ne se déclenche pas automatiquement → le problème est presque toujours la `description` (trop vague ou sans les mots-clés que l'utilisateur dirait naturellement). Fix : rendre la description plus spécifique et inclure les phrases déclencheurs. Les descriptions > 250 caractères sont tronquées dans la liste de skills chargée en contexte.

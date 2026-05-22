# Changelog

Toutes les modifications notables de ce projet sont documentées dans ce fichier.

Le format est basé sur [Keep a Changelog](https://keepachangelog.com/fr/1.1.0/),
et ce projet adhère au [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.7.1] - 2026-05-22

### Changed
- Documentation : extraction de l'inventaire des skills dans des fichiers dédiés `documentation/<plugin>.md` (un fichier par plugin, tableau 2 colonnes skill / rôle). Le `README.md` ne contient plus que le tableau des plugins disponibles enrichi d'une colonne « Inventaire » pointant vers chaque fichier, et la version du plugin `workflow` est passée de `0.18.0` à `0.20.0` (alignement avec `marketplace.json`).
- `CLAUDE.md` : ajout de `documentation/<plugin>.md` à la source de vérité et intégration de la maintenance de ces fichiers dans les procédures « ajouter une skill » et « créer un nouveau plugin ».

## [1.7.0] - 2026-05-22

### Added
- Plugin `workflow` : agent `autopilot` pour piloter en autonomie les skills `/workflow:feature`, `/workflow:refactor`, `/workflow:tech` — délègue chaque sous-tâche à un sous-agent isolé (contexte propre, scalable), trace l'avancement dans `.autopilot.json` (reprise possible après interruption), s'arrête uniquement aux stop-points stratégiques (verrou caractérisation, baseline mesurée, écart majeur détecté, avant tests finaux). Critères mineur/majeur explicites pour gérer les déviations sans bruit inutile.
- Plugin `workflow` : agent `report-and-sync` pour enchaîner `/workflow:report` puis `/workflow:sync` en une passe — clôture documentaire complète d'une story livrée.
- Guide `/help` : nouvelle section « Agents (orchestrateurs multi-skills) » documentant `report-and-sync` et `autopilot` avec leur rôle, leur cas d'usage et l'exemple d'invocation via le tool `Agent`.

### Changed
- Skill `/workflow:commit` rendu autonome : commit et push sans validation interactive du message ni du push. Sync systématique par `git fetch` + `git rebase` avant push (zéro merge commit), conflit de rebase → arrêt sans auto-résolution, push sûr via `--force-with-lease` uniquement après rebase effectué dans la session. Garanties bloquantes resserrées sur secrets, debug, fichiers temporaires, `--amend` déjà publié, `--no-verify`, `--force` nu.
- Plugin `workflow` bumpé de `0.20.0` à `0.23.0` (cumul des releases 0.21.0 → 0.22.0 → 0.23.0 livrées dans cette version).

## [1.6.0] - 2026-05-22

### Changed
- Plugin `workflow` : retrait du `model:` pinné dans le frontmatter de tous les skills — Claude utilise désormais le modèle par défaut du contexte, ce qui rend le plugin plus flexible pour les utilisateurs et leurs harnesses
- Guide `/help` : ajout des skills `doc-feature` et `release` (oubliés du sommaire), nouvelle section « Clôture de track » détaillant `/commit`, `/report`, `/sync` et clarification de la complémentarité `/report` vs `/sync`
- Plugin `workflow` bumpé de `0.18.0` à `0.20.0`

## [1.5.0] - 2026-05-21

### Added
- Skill `workflow:adr` — atelier interactif de rédaction d'Architecture Decision Records au format MADR léger (Contexte, Decision drivers, Options considérées, Décision, Conséquences, Links) → `docs/adr/NNNN-<slug>.md`. Trois modes d'entrée : depuis un artifact existant (`design.md`, `plan.md`, `review.md`, `report.md`), depuis un slug de story, ou depuis un topic libre. Phase d'exploration du code et du contexte, challenge minimum de 2 options sérieuses par décision, gestion du statut (`proposed` / `accepted` / `superseded`).
- Backlinks automatiques : ajout d'une ligne `> ADR :` dans l'artifact source, mise à jour de l'index `docs/adr/README.md` (table triée par numéro avec statut et story liée), section "Décisions architecturales" dans le `report.md` de la story si présent. Gestion du `superseded` en édition croisée de l'ancien ADR.
- Template `references/template.md` chargé à la demande pour rester en progressive disclosure (SKILL.md d'environ 12 K, template d'environ 3 K).

### Changed
- Skill `workflow:help` — schéma utilitaires et tableau mis à jour avec `/adr`.
- Plugin `workflow` bumpé `0.17.0` → `0.18.0`, synchronisé dans `marketplace.json` et `README.md` (table plugins + inventaire 19 → 20 skills).

## [1.4.1] - 2026-05-20

### Changed
- Skills `workflow` : externalisation du contenu volumineux (templates, cheatsheets, procédures conditionnelles) vers `references/*.md` chargés à la demande — SKILL.md réduits de 2508 → 1503 lignes (-40 % sur 9 skills), tokens à l'invocation diminués d'autant
- Frontmatter normalisé sur les 19 skills `workflow` : descriptions reformulées (verbes à l'indicatif, résumé orienté usage), ajout systématique de `model:` (haiku/sonnet/opus selon la skill) et `allowed-tools:` explicite, `disable-model-invocation: true` sur les skills user-only
- Plugin `workflow` bumpé `0.15.0` → `0.17.0`, synchronisé dans `marketplace.json` et `README.md`

## [1.4.0] - 2026-05-20

### Added
- Skills `workflow:vision` et `workflow:product-backlog` — quatre modes explicites (Création, Enrichir, Éditer, Pivot) demandés via `AskUserQuestion` quand le document existe déjà, pour éviter une session marathon quand on ne touche qu'un axe
- Phase de ciblage dédiée pour les modes Enrichir/Éditer (Phase 1bis vision, Phase 0bis backlog) — saute l'atelier complet et se concentre sur le delta
- Contrôles de cohérence systématiques avant rédaction : alignement vision, conflit anti-objectifs, rattachement, doublon, trous laissés par retrait, bascule auto vers Pivot si l'évolution dérive
- Archivage de l'ancien fichier sous `*.archive-AAAA-MM-JJ` en mode Pivot

### Changed
- Descriptions des skills `vision` et `product-backlog` enrichies de phrases déclencheurs adaptées aux nouveaux modes (« ajouter une audience à la vision », « enrichir le backlog », etc.)
- Plugin `workflow` bumpé `0.14.0` → `0.15.0` — synchronisé dans `marketplace.json`

## [1.3.0] - 2026-05-14

### Added
- Skill `workflow:product-backlog` — **Phase 0.5** du pipeline : atelier de cadrage du périmètre fonctionnel qui traduit `docs/vision.md` en domaines, capacités, parcours utilisateur, règles transverses et backlog priorisé MVP/V2/V3 → `docs/product-backlog.md`. Document vivant lu par `feature-pitch` pour situer chaque feature dans le périmètre et reprendre son pitch initial.

### Changed
- Skill `workflow:vision` — propose désormais `/product-backlog` comme étape suivante naturelle après la vision
- Skill `workflow:feature-pitch` — lit `docs/product-backlog.md` s'il existe pour récupérer le pitch initial d'une ligne backlog, ses capacités couvertes, ses dépendances et son alignement vision avant de challenger
- Skill `workflow:help` — sommaire et diagramme du pipeline mis à jour avec la phase 0.5
- Plugin `workflow` bumpé à `0.14.0` (ajout `product-backlog`) — synchronisé dans `marketplace.json`
- README enrichi d'un tutoriel détaillé du pipeline workflow (philosophie, carte mentale, tour des skills par track, exemple de bout en bout)

## [1.2.0] - 2026-05-09

### Added
- Skill `workflow:doc-feature` — généralisation stack-agnostique de l'ancien `sylius:doc-sylius`. Documente une feature déjà implémentée dans n'importe quel projet (PHP, Symfony, Sylius, autre) → `docs/feature-map/NNN-slug/feature.md`. Détection automatique du stack et chargement de la référence appropriée (`references/sylius.md`, `references/symfony.md`).

### Removed
- BREAKING : Skill `sylius:doc-sylius` — remplacée par `workflow:doc-feature` qui couvre le cas Sylius via `references/sylius.md` (Tabler, ThemeAlpha/Beta/TailwindTheme, Twig Hooks, grids, workflows, JWT, multi-channel) et fonctionne sur tout autre stack.

### Changed
- Plugin `workflow` bumpé à `0.13.0` (ajout `doc-feature`), plugin `sylius` bumpé à `0.26.0` (retrait `doc-sylius`) — synchronisés dans `marketplace.json`

## [1.1.0] - 2026-05-08

### Changed
- 20 descriptions de skill resserrées sous le seuil de 250 caractères (au-delà, la description est tronquée dans la liste de skills chargée en contexte) en préservant les phrases de déclenchement
- 9 skills Sylius enrichis d'une section « Déclenche sur… » pour améliorer leur invocation automatique : `doc-sylius`, `dynamic`, `email`, `fixtures`, `model`, `styles`, `template`, `translation`, `validation`
- Plugins bumpés : `workflow` 0.12.0, `editorial` 0.3.0, `sylius` 0.25.0 — synchronisés dans `marketplace.json`

## [1.0.0] - 2026-05-08

### Notes

Première version stable de la marketplace `gabrielmustiere`. Le format des plugins et le namespacing des skills sont désormais figés. **63 skills** réparties sur 4 plugins thématiques :

- `workflow` — 17 skills (pipeline de développement stack-agnostique)
- `symfony` — 25 skills (Symfony / Doctrine par domaine)
- `sylius` — 18 skills (e-commerce Sylius 2.x)
- `editorial` — 3 skills (rédaction d'articles & side-projects)

### Added
- Skill `workflow:vision` — **Phase 0** du pipeline : atelier de cadrage de la vision projet (problème, audience, valeur, North Star, principes, anti-objectifs) → `docs/vision.md`. Document fondateur lu par `feature-pitch` pour challenger l'alignement de chaque feature.

### Changed
- README focalisé sur l'utilisateur final (installation + inventaire des skills) — sections de contribution retirées
- `marketplace.json` synchronisé avec `plugin.json` pour le plugin `workflow` (0.11.0)
- `CLAUDE.md` et `README.md` alignés sur la nomenclature `workflow` (résidus `dev-workflow` purgés) et la nouvelle phase 0 vision

## [0.7.0] - 2026-04-29

### Added
- Skill `editorial:article-rework` pour retoucher chirurgicalement une portion d'article publié (chapitre, section, paragraphe) — respect de la voix de l'article, lecture du `plan.md` associé, mise à jour du plan si la promesse d'une section change, propagation à la traduction si une version existe, vérifications schéma + lint + format

### Changed
- Plugin `editorial` bumpé à `0.2.0` — description enrichie pour refléter le workflow complet en trois étapes (article-plan → article → article-rework)
- `marketplace.json` et `README.md` synchronisés à `0.2.0` pour le plugin `editorial` (catalogue + inventaire des skills)

## [0.6.0] - 2026-04-27

### Added
- Skill `workflow:import-external` pour migrer des spécifications produites avec Spec Kit, BMAD-METHOD ou GSD vers le format `docs/story/NNN-<f|r|t>-<slug>/`
- Skill `workflow:migrate-legacy` pour renommer les anciens dossiers `docs/story/<f|r|t>-NNN-<slug>/` au nouveau format avec compteur en tête, en préservant l'historique via `git mv`
- Skill `workflow:release` pour publier une version : analyse SemVer des commits, mise à jour du `CHANGELOG.md`, tag annoté et release GitHub

### Changed
- Plugin `workflow` synchronisé à `0.10.0` dans `marketplace.json` et `README.md` (alignement avec `plugin.json`)
- Inventaire workflow du `README.md` complété avec les skills `migrate-legacy`, `import-external` et `release`

[Unreleased]: https://github.com/gabrielmustiere/skills/compare/v1.7.1...HEAD
[1.7.1]: https://github.com/gabrielmustiere/skills/compare/v1.7.0...v1.7.1
[1.7.0]: https://github.com/gabrielmustiere/skills/compare/v1.6.0...v1.7.0
[1.6.0]: https://github.com/gabrielmustiere/skills/compare/v1.5.0...v1.6.0
[1.5.0]: https://github.com/gabrielmustiere/skills/compare/v1.4.1...v1.5.0
[1.4.1]: https://github.com/gabrielmustiere/skills/compare/v1.4.0...v1.4.1
[1.4.0]: https://github.com/gabrielmustiere/skills/compare/v1.3.0...v1.4.0
[1.3.0]: https://github.com/gabrielmustiere/skills/compare/v1.2.0...v1.3.0
[1.2.0]: https://github.com/gabrielmustiere/skills/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/gabrielmustiere/skills/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/gabrielmustiere/skills/compare/v0.7.0...v1.0.0
[0.7.0]: https://github.com/gabrielmustiere/skills/compare/v0.6.0...v0.7.0
[0.6.0]: https://github.com/gabrielmustiere/skills/compare/v0.5.0...v0.6.0
[0.5.0]: https://github.com/gabrielmustiere/skills/releases/tag/v0.5.0

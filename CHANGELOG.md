# Changelog

Toutes les modifications notables de ce projet sont documentées dans ce fichier.

Le format est basé sur [Keep a Changelog](https://keepachangelog.com/fr/1.1.0/),
et ce projet adhère au [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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

[Unreleased]: https://github.com/gabrielmustiere/skills/compare/v1.4.0...HEAD
[1.4.0]: https://github.com/gabrielmustiere/skills/compare/v1.3.0...v1.4.0
[1.3.0]: https://github.com/gabrielmustiere/skills/compare/v1.2.0...v1.3.0
[1.2.0]: https://github.com/gabrielmustiere/skills/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/gabrielmustiere/skills/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/gabrielmustiere/skills/compare/v0.7.0...v1.0.0
[0.7.0]: https://github.com/gabrielmustiere/skills/compare/v0.6.0...v0.7.0
[0.6.0]: https://github.com/gabrielmustiere/skills/compare/v0.5.0...v0.6.0
[0.5.0]: https://github.com/gabrielmustiere/skills/releases/tag/v0.5.0

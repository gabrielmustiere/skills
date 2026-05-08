# Changelog

Toutes les modifications notables de ce projet sont documentées dans ce fichier.

Le format est basé sur [Keep a Changelog](https://keepachangelog.com/fr/1.1.0/),
et ce projet adhère au [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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

[Unreleased]: https://github.com/gabrielmustiere/skills/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/gabrielmustiere/skills/compare/v0.7.0...v1.0.0
[0.7.0]: https://github.com/gabrielmustiere/skills/compare/v0.6.0...v0.7.0
[0.6.0]: https://github.com/gabrielmustiere/skills/compare/v0.5.0...v0.6.0
[0.5.0]: https://github.com/gabrielmustiere/skills/releases/tag/v0.5.0

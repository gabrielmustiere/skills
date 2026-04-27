# Changelog

Toutes les modifications notables de ce projet sont documentées dans ce fichier.

Le format est basé sur [Keep a Changelog](https://keepachangelog.com/fr/1.1.0/),
et ce projet adhère au [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.6.0] - 2026-04-27

### Added
- Skill `workflow:import-external` pour migrer des spécifications produites avec Spec Kit, BMAD-METHOD ou GSD vers le format `docs/story/NNN-<f|r|t>-<slug>/`
- Skill `workflow:migrate-legacy` pour renommer les anciens dossiers `docs/story/<f|r|t>-NNN-<slug>/` au nouveau format avec compteur en tête, en préservant l'historique via `git mv`
- Skill `workflow:release` pour publier une version : analyse SemVer des commits, mise à jour du `CHANGELOG.md`, tag annoté et release GitHub

### Changed
- Plugin `workflow` synchronisé à `0.10.0` dans `marketplace.json` et `README.md` (alignement avec `plugin.json`)
- Inventaire workflow du `README.md` complété avec les skills `migrate-legacy`, `import-external` et `release`

[Unreleased]: https://github.com/gabrielmustiere/skills/compare/v0.6.0...HEAD
[0.6.0]: https://github.com/gabrielmustiere/skills/compare/v0.5.0...v0.6.0
[0.5.0]: https://github.com/gabrielmustiere/skills/releases/tag/v0.5.0

# Inventaire — plugin `editorial`

Pipeline de rédaction d'articles de blog et fiches side-project (3 skills).

| Skill | Rôle |
| --- | --- |
| [`article-plan`](../plugins/editorial/skills/article-plan/SKILL.md) | Atelier de cadrage d'un article de blog ou d'une fiche side-project — sujet, thèse, audience, recherche, chapitrage, tonalité, frontmatter prévisionnel adapté à la stack détectée → `docs/story/a-<NNN>-<slug>/plan.md` |
| [`article`](../plugins/editorial/skills/article/SKILL.md) | Rédaction guidée à partir du `plan.md` validé — produit le fichier final dans la collection détectée (Astro CC, Next MDX, Hugo, Jekyll, markdown brut), frontmatter conforme au schéma, vérifications schéma + lint + format, traduction multilingue si prévue |
| [`article-rework`](../plugins/editorial/skills/article-rework/SKILL.md) | Retouche chirurgicale d'une portion d'un article publié (chapitre, section, paragraphe) — respecte la voix de l'article, lit le `plan.md` associé, met à jour le plan si la promesse de la section change, propage à la traduction, vérifie schéma + lint + format |

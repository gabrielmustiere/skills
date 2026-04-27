---
name: article-plan
description: Cadrage d'un article de blog ou side-project AVANT rédaction — sujet, thèse, audience, recherche, chapitrage, tonalité, frontmatter adapté à la stack (Astro CC, Next.js MDX, Hugo, Jekyll, markdown brut). Produit `docs/story/a-<NNN>-<slug>/plan.md`. Première étape avant le skill `article`. Déclenche sur "j'ai une idée d'article", "écrire un billet sur X", "fais-moi un plan d'article", "structure cet article", "side-project à documenter", "chapitre un article", "thèse pour un article", "j'écris sur…". À utiliser dès qu'un sujet à écrire est mentionné, avant toute rédaction directe.
metadata:
  version: 0.1.0
---

# Article Plan

Atelier de cadrage pour la **première étape** du workflow de rédaction d'un article (blog) ou d'une fiche side-project. Ne rédige **pas** l'article — produit un plan détaillé que le skill `article` exécutera ensuite.

Le plan vit dans `docs/story/a-<NNN>-<slug>/plan.md`, dans la lignée des dossiers `f-` (feature), `r-` (refacto), `t-` (tech) du plugin `workflow`. `a-` pour *article*.

## Pourquoi ce skill existe

Un article qui rate sa cible rate toujours pour la même raison : **angle flou, audience indistincte, thèse qui n'avance rien**. La rédaction directe sans plan masque ces défauts derrière de la fluidité. Ce skill force la décision avant l'écriture : *qu'est-ce que ce texte change pour le lecteur ?*. Une fois cette décision prise, écrire devient mécanique.

Le plan est aussi le contrat passé à l'étape suivante (rédaction). Il doit être suffisamment précis pour qu'un rédacteur — humain ou agent — puisse écrire sans avoir à redécouvrir l'angle.

## Mode : blog vs side-project

Demander explicitement au début, sauf si évident depuis le brief :

- **Article de blog** → cible le dossier de contenus blog du projet (détecté en étape 0). Le frontmatter dépend de la stack détectée.
- **Side-project** → cible un dossier de fiches projet (souvent `src/content/projects/`, `content/projects/`, `_projects/`, etc.). Plan plus orienté narration produit (problème, ce qu'on a fait, résultat, leçons) que démonstration.

Les deux modes partagent la même structure de plan ; les différences sont marquées dans le template.

## Workflow

### 0. Pré-requis : détecter la stack et l'environnement

Avant de poser la moindre question éditoriale, **lire le projet** pour comprendre comment ses contenus sont organisés. C'est la garantie que le plan produit s'aligne sur les contraintes réelles du repo, pas sur des suppositions.

#### 0.1 Détection de stack

Vérifier la présence des marqueurs suivants, dans l'ordre. La première stack détectée gagne :

1. **Astro Content Collections** — `astro.config.mjs` ou `astro.config.ts` à la racine, et `src/content.config.ts` (ou `src/content/config.ts`) qui exporte des `defineCollection`. Lire le schéma Zod : il donne directement les bornes (`min`, `max`, `enum`), les champs requis, les valeurs par défaut. C'est la source de vérité. Ne pas la paraphraser — la lire.
2. **Next.js MDX / contentlayer** — `next.config.js`/`next.config.mjs` + un dossier `content/`, `posts/`, ou `src/posts/`. Regarder s'il y a `contentlayer.config.ts` (schéma typé) ou si les articles ont juste un frontmatter ad hoc.
3. **Hugo** — `hugo.toml`/`hugo.yaml`/`config.toml` à la racine + `content/posts/` ou équivalent.
4. **Jekyll** — `_config.yml` + `_posts/`.
5. **Markdown brut** — aucun framework détecté. Chercher un dossier `posts/`, `articles/`, `blog/`, `content/`, `docs/blog/`. Si aucun, demander.

Annoncer la détection au user en une ligne (« j'ai détecté une stack Astro Content Collections, schéma blog dans `src/content.config.ts` ») — ça lui permet de corriger immédiatement si la détection est fausse.

#### 0.2 Lecture des conventions

Une fois la stack détectée :

- **Lister les articles existants** (FR + EN si bilingue, voir 0.3) pour connaître la collection cible.
- **Lire 1 à 2 articles récents en entier** — pas juste le frontmatter. La voix vit dans le corps. Repérer : longueur des phrases, fréquence des listes, ratio code/prose, voix (je / on / impersonnel), niveau de jargon assumé, présence ou non de TL;DR, conventions de titres, tics récurrents (ouverture par anecdote, citations de fichiers avec chemin, etc.).
- **Si plusieurs catégories ou tags** existent, en faire l'inventaire — c'est utile pour calibrer le frontmatter prévisionnel sans imposer une catégorie nouvelle quand une existante colle.

#### 0.3 Détection multilingue

Indices de site bilingue / multilingue :

- Astro : champ `lang` dans le schéma de collection, dossier `src/pages/<lang>/`, fichier `astro.config.mjs` avec config `i18n`.
- Next : dossier `app/[lang]/` ou `pages/[lang]/`, ou config `next-i18next`.
- Hugo : section `[languages]` dans la config.
- Jekyll : plugin `jekyll-multiple-languages` ou structure `_posts/<lang>/`.
- Champs `translationOf`, `lang`, `locale` dans les frontmatters existants.

Si bilingue détecté : noter quelle langue est la principale (souvent celle sans préfixe d'URL ou avec le plus d'articles), et quelle est la cible de traduction. Demander au user en cas de doute.

Si monolingue : ne pas inventer de stratégie de traduction. Le plan ne doit pas mentionner de pendant linguistique.

#### 0.4 Numérotation et compteurs

- **Lister `docs/story/a-*`** pour déterminer le prochain `NNN` du **dossier de plan** (incrément simple, sur 3 chiffres : `a-001`, `a-002`...). C'est la convention du plugin `workflow`. Si `docs/story/` n'existe pas, le créer.
- **Si la stack a un champ `number`** dans le schéma (Astro CC notamment), c'est un compteur **séparé** du dossier de plan. Il reflète l'ordre de publication dans la collection ; il s'incrémente en lisant les frontmatters publiés. Pour une collection bilingue où une paire FR/EN partage le même `number` (via `translationOf`), filtrer par langue principale pour calculer la séquence. Sinon, prendre `max(number) + 1`.
- Si le schéma n'a pas de `number`, ne pas l'inventer.

#### 0.5 Brief partiel ?

Si un brief substantiel a déjà été donné dans la conversation, **extraire** ce qui est déjà répondu — ne pas reposer ces questions. Récapituler ce qu'on a compris et demander juste les manques.

### 1. Cadrage initial — l'atelier

Poser les questions par paquets cohérents, **pas une par une**. Idéalement deux tours de questions, pas plus. Utiliser `AskUserQuestion` quand des choix discrets sont possibles (catégorie, format, longueur cible) ; texte libre sinon.

**Tour 1 — sujet & cible :**

- **Type** : article de blog ou side-project ? (Sauter si évident depuis le brief.)
- **Sujet en une phrase** : de quoi ça parle ? (pas le titre — la matière)
- **Pourquoi maintenant** : déclencheur concret (lecture, mission récente, frustration, lancement). Sans déclencheur clair, l'article risque d'être tiède.
- **Audience cible** : qui doit lire ça ? (CTO freelance ? dev senior curieux d'IA ? recruteur tech ?). Une seule audience principale — pas trois.
- **Thèse en une phrase** : qu'est-ce que le lecteur **doit avoir compris ou changé d'avis** à la fin ? Si la réponse est « comprendre X », c'est insuffisant — pousser jusqu'à *« comprendre que X plutôt que Y »*.

**Si la thèse revient molle** (formats déguisés en thèse : « avantages/inconvénients », « tour d'horizon », « comprendre X », « présentation de Y ») : ne pas reposer la question dans le vide — c'est inefficace. À la place, **proposer 3 ou 4 thèses candidates calibrées sur le déclencheur** que le user a donné, et le laisser choisir / amender / dire « Other ». Une bonne thèse candidate prend la forme `« X plutôt que Y, parce que Z »` ou `« X est l'aboutissement quand W »` ou `« X règle le problème Y mieux que Z »`. Le user reconnaît plus facilement la bonne thèse qu'il ne la formule.

**Tour 2 — sources & forme :**

- **Sources fournies** : URLs, docs internes du repo (CLAUDE.md, articles existants, code), notes. Le skill les lit.
- **Recherche web active souhaitée ?** Par défaut : oui sur sujets tech/IA qui datent vite, non sur sujets evergreen. Annoncer explicitement les requêtes prévues avant de les lancer.
- **Référence de ton** : pointer 1–2 articles existants du repo que le user veut prolonger en style. Si aucun n'est cité, prendre l'article le plus récent comme proxy.
- **Format & longueur visée** : article court (1500–2500 mots), long (3000–5000), ou autre. Pour un side-project, généralement plus court (800–1500).
- **Catégorie pressentie** (si la stack en a une, ex. enum Astro `category`) : présenter les catégories existantes du projet pour que le user choisisse parmi elles plutôt que d'en inventer.

### 2. Recherche

Une fois les sources connues :

- **Lire les sources fournies** (Read sur les fichiers, WebFetch sur les URLs). Noter les passages utilisables.
- **Lire 1–2 articles existants** pour calibrer le ton (déjà détectés en 0.2). Ne pas refaire le travail si c'est déjà couvert.
- **WebSearch ciblé** si activé : 2–4 requêtes maximum, focalisées sur (a) angles concurrents déjà publiés (pour différencier), (b) données chiffrées récentes, (c) terminologie consensuelle. Annoncer chaque requête. Ne pas accumuler 15 résultats — extraire les 3–5 angles dominants.
- **Synthèse de recherche** : 5–10 lignes max résumant ce qui ressort, ce qui manque, l'angle libre à occuper. Cette synthèse va dans le plan.

### 3. Cadrage de l'angle (checkpoint humain)

Avant d'écrire le plan détaillé, **valider** avec le user :

- L'angle unique en une phrase (« je raconte X **du point de vue de** Y, en montrant que Z »).
- Ce que l'article **n'est pas** (pour éviter le scope creep). Ex : « ce n'est pas un tuto Astro, ce n'est pas une comparaison Astro/Next ».
- L'effet attendu chez le lecteur (action, changement de modèle mental, partage).

Si le user n'est pas convaincu, itérer ici — pas plus loin. Un mauvais angle ne se rattrape pas par un beau chapitrage.

### 4. Chapitrage proposé

Proposer une structure et la **soumettre à validation** avant de figer dans le plan. Préférer 4–7 sections principales pour un article moyen, 3–4 pour un side-project. Pour chaque section :

- **Titre** (style éditorial, pas SEO bourrin) — court, idéalement ≤ 60 caractères.
- **Promesse** : 1–2 lignes de ce que la section apporte.
- **Points clés à couvrir** : 3–6 bullets de matière (pas de prose).
- **Artefacts** : code, tableau, citation, lien — ce qui doit y figurer concrètement.

Présenter le chapitrage en bloc. Inviter explicitement à supprimer / réordonner / fusionner. Une section « Conclusion » générique est un signal d'angle faible — préférer une section nommée qui *dit* quelque chose.

### 5. Tonalité & voix

Décrire en 5–8 lignes :

- **Voix** : je / on / impersonnel. Calquer sur la voix dominante détectée dans les articles existants (étape 0.2). Si le user veut s'en écarter, l'expliciter.
- **Niveau** : praticien (assume le jargon) ou pédagogique (explique). Mixte est possible si bien jalonné.
- **Rythme** : phrases courtes / longues, équilibre paragraphes / listes / blocs de code.
- **Tics à éviter** : exclamations, emojis, « dans cet article nous verrons », relances LinkedIn-like, MAJUSCULES emphatiques, em-dashes en pagaille (un par paragraphe max).
- **Tics à reproduire** : repris du ou des articles de référence — citer 1–2 marqueurs (« ouvre par une situation concrète », « cite des fichiers avec leur chemin », etc.).

### 6. Frontmatter prévisionnel

Pré-remplir, en respectant **strictement** le schéma détecté en étape 0. Si le schéma a des bornes (min/max), les contraindre dès le plan — c'est gratuit ici, douloureux plus tard.

**Pour une stack avec schéma typé (Astro CC, contentlayer)** : reprendre les champs du schéma à la lettre, proposer 2–3 variantes pour les champs ouverts (`title`, `excerpt`, `tldr`), respecter les enums (`category`, `status`).

**Pour une stack à frontmatter libre** (Hugo / Jekyll / markdown brut) : se caler sur le frontmatter des articles existants. Si tous ont `title`, `date`, `tags`, garder ce trio. Ne pas inventer de champs absents de la convention courante — un nouvel article doit ressembler aux précédents.

**Champs recommandés à anticiper, même hors schéma typé :**

```yaml
title: "..."           # propose 2–3 variantes
excerpt: "..."         # 80–220 chars en règle générale (à adapter au schéma détecté)
date: <YYYY-MM-DD>
tags: [...]
keywords: [...]        # si la stack les distingue des tags
lang: <code>           # si bilingue
slug: <kebab-case>
```

**Stratégie multilingue :** si le projet est bilingue (détecté en 0.3) **et** si une version traduite est attendue pour cet article, anticiper :

- Le slug de la version traduite (souvent : traduire le slug, pas suffixer `-en`).
- Les champs `lang`/`locale` et le pointeur de traduction (`translationOf`, `alternate`, ou la convention détectée dans les articles bilingues existants).
- Si la stack utilise un compteur partagé entre versions linguistiques, le mentionner explicitement dans le plan.
- Décision de traduire à confirmer **après** validation de la version source — pas en parallèle. Mettre dans le plan : *« Pendant traduit : prévu / pas prévu / à décider après source validée »*.

Si le projet est monolingue, **ne pas mentionner de stratégie de traduction**. Sauter cette sous-section.

### 7. Risques & garde-fous

Lister 3–5 pièges propres à ce sujet :

- Claim non-sourçable → marquer « à étayer ou retirer ».
- Section qui sent le bourrage SEO → marquer « à couper si rédaction tire vers le mot-clé ».
- Tendance à dériver vers un autre sujet (ex: parle de migration mais déborde sur SEO) → marquer « limite explicite ».
- Bench / chiffre qui peut périmer en 6 mois → noter la date de validité.

Ces garde-fous protègent l'étape de rédaction.

### 8. Génération du fichier `plan.md`

Une fois validé, écrire `docs/story/a-<NNN>-<slug>/plan.md` en suivant exactement le template ci-dessous. Le slug est dérivé du sujet en kebab-case, court (≤ 6 mots), sans accents — il deviendra typiquement le slug final du fichier.

## Template du fichier `plan.md`

Reproduire à l'identique, en remplissant les champs. **Garder les sections vides explicites** plutôt que de les supprimer — un placeholder `_à compléter en rédaction_` est préférable au silence.

```markdown
# Plan d'article — <titre de travail>

> Date : <YYYY-MM-DD>
> Type : article-blog | side-project
> Slug pressenti : <slug>
> Stack détectée : <Astro CC | Next MDX | Hugo | Jekyll | markdown>
> Langue source : <code>
> Multilingue : <oui/non — si oui, langues cibles>

## Sujet & déclencheur

<2–4 lignes : de quoi ça parle, pourquoi maintenant>

## Audience

<1–2 lignes : qui, niveau, attentes>

## Thèse

<une phrase : le lecteur doit avoir compris/changé d'avis sur __>

## Angle

<une phrase : "je raconte X du point de vue de Y, en montrant Z">

**Ce n'est pas :**
- <hors-périmètre 1>
- <hors-périmètre 2>

## Synthèse de recherche

**Sources lues :**
- <chemin ou URL> — <ce qu'on en garde>

**Recherches web :**
- <requête> → <ce qui ressort>

**Angles concurrents identifiés :** <liste courte>
**Angle libre à occuper :** <phrase>

## Chapitrage

### 1. <Titre de section>
**Promesse :** <1–2 lignes>
**Points clés :**
- <bullet>
- <bullet>
**Artefacts :** <code / tableau / lien / citation>

### 2. <Titre de section>
...

## Tonalité

- **Voix :** je | on | impersonnel
- **Niveau :** praticien | pédagogique | mixte
- **Rythme :** <descriptif court>
- **À éviter :** <liste>
- **À reproduire :** <marqueurs depuis articles de référence>

**Articles de référence :**
- <chemin>

## Frontmatter prévisionnel

\`\`\`yaml
<bloc YAML adapté au schéma détecté en étape 0>
\`\`\`

**Variantes de titre envisagées :**
1. <variante>
2. <variante>
3. <variante>

**Pendant traduit :** prévu | pas prévu | à décider après source validée | sans objet (monolingue)
<si prévu : slug traduit pressenti, pointeur de traduction à poser, compteur partagé éventuel>

## Risques & garde-fous

- <risque> → <garde-fou>
- <risque> → <garde-fou>

## Prochaine étape

Rédaction complète à partir de ce plan. Lancer le skill `article` (`/editorial:article`) ou demander : « rédige l'article depuis ce plan ».
```

## Conventions et garde-fous du skill

- **Ne jamais écrire le corps de l'article ici.** Tentation forte : la résister. Le plan doit s'arrêter au niveau bullets/promesses. Si le user demande « écris-le maintenant », confirmer que le plan est validé puis pointer vers l'étape `article` (ou rédiger explicitement, hors workflow).
- **Respecter les contraintes du schéma détecté** — bornes Zod, enums, champs requis. Un plan qui propose un `excerpt` hors bornes est cassé d'office.
- **Pas d'invention de chiffres.** Si une donnée n'a pas de source, marquer « _à étayer en rédaction_ » plutôt que de poser un chiffre fragile.
- **Slug court, sans accents, kebab-case.** Cohérent avec les fichiers existants de la collection.
- **Numérotation `a-NNN`** : strictement incrémentale, jamais réutiliser. Padding 3 chiffres. C'est la convention `docs/story/` du plugin `workflow`.
- **Si un `docs/story/a-<NNN>-<slug>/plan.md` existe déjà** pour un sujet proche, le proposer en lecture avant de créer un nouveau dossier — éviter les plans orphelins.
- **Recherche web : annoncer avant, citer après.** Chaque source consultée apparaît dans la section « Sources lues » du plan, avec son URL.
- **Stack détectée mais user en désaccord** : la détection n'est qu'une hypothèse. Si le user dit « non, on est en Hugo pas en Jekyll », recaler immédiatement et adapter la suite. Ne pas insister.

## Exemples d'amorçage

**Exemple 1 — brief riche, stack typée :**

User : « Je veux écrire un article sur comment Claude Code change la façon de bosser pour un CTO freelance, j'ai mes notes dans `docs/notes/cto-claude.md` et je veux le ton de l'article `building-this-site-with-claude-and-astro`. »

→ Détecter la stack (Astro CC), lire le schéma Zod, lire l'article de référence, repérer la voix « je ». Sauter le tour 1 (audience, déclencheur, thèse explicites dans le brief), faire 1–2 requêtes web pour les angles concurrents, passer direct au cadrage d'angle.

**Exemple 2 — brief vague, stack à confirmer :**

User : « J'ai un side-project autour de l'observabilité avec OpenTelemetry, je veux le documenter. »

→ Détecter la stack et la collection cible. Si plusieurs candidates (`projects/`, `content/projects/`), demander. Tour 1 obligatoire (audience, thèse, déclencheur — un side-project sans thèse devient un README). Demander si la fiche est destinée à attirer du recrutement, à pédagogiser un pattern, ou à archiver une expé.

**Exemple 3 — sujet déjà commencé :**

User : « Reprends le plan dans `docs/story/a-002-rag-vs-finetune/plan.md`, j'ai des nouvelles sources. »

→ Lire le plan existant, demander les nouvelles sources, mettre à jour la section recherche et chapitrage uniquement, ne pas refaire l'atelier complet. Re-vérifier la détection de stack en passant — un projet peut avoir migré entre temps.

**Exemple 4 — stack non détectée :**

User : « Fais-moi un plan d'article sur les feature flags. »

→ Aucune stack standard détectée. Demander explicitement : « Où vivent tes articles ? » + extension attendue (`.md` / `.mdx`) + frontmatter requis (montrer un exemple existant si possible, sinon partir d'un frontmatter minimal `title` + `date` + `tags`). Dérouler le workflow normal ensuite.

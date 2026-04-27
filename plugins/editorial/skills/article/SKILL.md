---
name: article
description: Rédaction guidée d'un article de blog ou side-project à partir du `plan.md` validé sous `docs/story/a-<NNN>-<slug>/` — produit le fichier final dans la collection détectée (Astro CC, contentlayer, Hugo, Jekyll, markdown brut), frontmatter conforme au schéma, vérifications schéma + lint + format, traduction multilingue si prévue. Deuxième étape après `article-plan`. Déclenche sur "rédige l'article depuis ce plan", "écris l'article a-002", "passe à la rédaction", "exécute le plan d'article", "draft l'article", "rédige depuis docs/story/a-…". À utiliser dès qu'un `plan.md` existe et que le user veut passer à l'écriture.
metadata:
  version: 0.1.0
---

# Article

Exécution guidée d'un plan d'article (ou de side-project) déjà validé. Le plan est le contrat ; ce skill ne le ré-ouvre pas — il l'exécute.

Entrée : `docs/story/a-<NNN>-<slug>/plan.md`.
Sortie : un fichier dans la collection détectée du projet (par exemple `src/content/blog/<slug>.mdx`, `content/posts/<slug>.md`, `_posts/<date>-<slug>.md`...), frontmatter conforme au schéma de la stack, build vert, lint propre, et — si prévu dans le plan — la version traduite à la suite.

## Pourquoi ce skill existe

Une rédaction sans plan dérive ; une rédaction qui rejoue le cadrage à chaque section perd l'élan. Ce skill suppose que le travail conceptuel a été fait (`article-plan`) et se concentre sur **deux risques propres à la rédaction** :

1. **La voix qui dérape.** L'intro fixe la voix de tout l'article. Si la voix n'est pas calée à la fin de la section 1, les 3000 mots qui suivent sont à refaire. → Checkpoint humain obligatoire **après l'intro + section 1**, jamais ailleurs.
2. **Le frontmatter qui casse le build.** Les schémas typés (Astro CC, contentlayer) sont stricts ; les conventions des stacks à frontmatter libre (Hugo / Jekyll / markdown) sont implicites mais tout aussi fragiles. Un brouillon qui ne valide pas n'est pas un brouillon, c'est un blocker. → Le frontmatter est **figé en fin de rédaction** et **vérifié** par les commandes du projet avant de rendre la main.

Tout le reste — cohérence des sections, respect du chapitrage, ton — découle du plan.

## Mode : blog vs side-project

Le mode est déjà fixé dans le plan (`Type : article-blog | side-project`). Lire cette ligne, ne pas reposer la question.

- **article-blog** → cible la collection blog du projet (chemin et extension détectés).
- **side-project** → cible la collection projects/portfolio du projet, souvent avec un schéma plus narratif.

Les deux modes partagent le workflow ; les différences se résolvent au moment d'écrire le frontmatter (étape 4).

## Pré-requis : auditer le plan ET le projet avant la première phrase

### Audit du plan

Lire intégralement le plan et **bloquer** si l'un de ces critères manque :

- `## Thèse` non vide et formulée comme une position (pas « comprendre X »).
- `## Angle` non vide.
- `## Chapitrage` avec au moins 3 sections, chacune avec **Promesse** + **Points clés**.
- `## Tonalité` renseignée (au minimum la voix et les articles de référence).
- `## Frontmatter prévisionnel` avec un titre proposé (au moins une variante).

Si l'un manque : ne pas combler à la place du user. Lui dire ce qui manque et proposer de **revenir au skill `article-plan`** pour itérer. Un plan incomplet est un piège — il invite à inventer l'angle pendant la rédaction, ce qui fabrique le défaut exact que `article-plan` doit prévenir.

### Audit du projet (stack & commandes)

Le plan note la `Stack détectée` ; recouper avec l'état actuel du repo (la stack peut avoir bougé depuis l'écriture du plan) :

1. **Re-détecter la stack** : Astro CC (`astro.config.*` + `src/content.config.ts`), Next MDX, contentlayer, Hugo, Jekyll, markdown brut. Mêmes critères qu'`article-plan` étape 0.
2. **Lire le schéma de contenu si typé** (Astro `src/content.config.ts`, `contentlayer.config.ts`) — il prime sur ce que dit le plan en cas de divergence (le schéma a peut-être changé).
3. **Identifier la collection cible exacte** (chemin + extension). Si plusieurs candidates, demander.
4. **Détecter les commandes de vérification** dans `package.json` (ou équivalent) :
   - **Schéma / typecheck** : `astro check`, `next build` / `next lint`, `tsc --noEmit`, `hugo --check`, ou rien (markdown brut).
   - **Lint** : `npm run lint`, `npm run lint:fix`, `eslint .`, `markdownlint`...
   - **Format** : `npm run format`, `prettier --write .`, `dprint fmt`...
   - Si un script personnalisé existe (ex. `npm run check`), le préférer aux invocations directes.

Lire aussi, **avant la rédaction** :

- Le ou les **articles de référence** cités dans `## Tonalité`. Lire le corps complet, pas juste le frontmatter — c'est là que la voix vit.
- L'article le plus récent de la collection cible (filtré par langue source du plan), pour caler la mise en forme courante (titres `h2`/`h3`, blocs de code, citations, listes, composants MDX éventuels).

## Workflow

### 0. Vérifier l'environnement et le plan

1. Identifier le plan ciblé : `docs/story/a-<NNN>-<slug>/plan.md`. Si plusieurs `a-*` existent et que le user n'a pas désigné lequel, lister les plans présents (titre + date + type) et demander.
2. Auditer le plan (voir section précédente). Si incomplet → renvoyer vers `article-plan`.
3. Auditer la stack et les commandes (voir section précédente).
4. Vérifier que **le slug du plan n'existe pas déjà** comme article publié dans la collection cible. Si oui : refuser d'écraser, demander au user (mise à jour ? nouveau slug ? abandon ?).

### 1. Cadrage rapide — 3 questions, pas plus

Avant d'écrire l'intro, confirmer :

- **Statut de publication** : draft (`draft: true` ou champ équivalent selon stack) ou publication immédiate ? Par défaut, **draft** sur la première version — on valide le rendu avant de basculer publié. Si le schéma n'a pas de champ `draft`, sauter ; sinon utiliser le nom détecté.
- **Date de publication** : aujourd'hui par défaut. Si le plan en propose une, la confirmer. Champ exact selon stack (`publishedAt`, `date`, `pubDate`, `year`...).
- **Cas applicatif et choix laissés ouverts dans le plan** : certains plans listent explicitement *« à figer en début de rédaction »* (ex : choix d'un cas applicatif, choix d'un titre parmi 3 variantes). Récapituler ces décisions ouvertes en une question groupée.

Utiliser `AskUserQuestion` pour les choix discrets (titre parmi N, draft true/false). Pas de tour de questions long ici — le plan a déjà tranché.

### 2. Rédiger l'intro + la section 1, puis CHECKPOINT

C'est l'étape la plus importante du skill. Écrire :

- **L'intro** (avant le premier `##`) : 2 à 4 paragraphes qui ouvrent par une situation concrète (jamais par une définition ni un *« dans cet article… »*), posent la thèse en mots simples, et donnent envie de continuer. Pas de meta-promesses (« nous verrons »). Pas de TL;DR dans le corps si la stack a un champ dédié — le frontmatter joue ce rôle ailleurs.
- **La section 1** complète, en suivant la **Promesse** et les **Points clés** du chapitrage, dans le ton fixé par les articles de référence.

Puis **arrêter** et présenter ces deux blocs au user, avec ce message explicite :

> *« Voici l'intro et la section 1. C'est ici que la voix se fixe pour les ~3000 mots qui suivent — relis ce passage avant que je ne continue. Dis-moi ce qui sonne juste, ce qui sonne faux, ce qu'il faut couper, ajouter, ou réécrire. Je n'enchaîne pas tant que tu n'as pas validé. »*

**Ne pas continuer** sans feedback explicite (« OK, continue », « parfait, suite », ou des modifs ciblées suivies d'un OK). Si le user demande de modifier, appliquer **puis re-checkpoint** — ne pas enchaîner sur la section 2 dans la foulée d'une réécriture de l'intro, parce que la voix vient juste d'être recalée.

Si le user dit *« écris tout et on verra »* : tenir bon. Le checkpoint après section 1 est la garantie principale du skill ; l'enjamber transforme le skill en générateur de bouillie. Expliquer brièvement pourquoi (la voix se fixe ici, refaire 3000 mots coûte plus cher que relire 500), puis re-proposer le checkpoint.

### 3. Rédiger les sections suivantes

Une fois l'intro validée :

- Rédiger **toutes les sections restantes en un seul jet**, dans l'ordre du chapitrage. Pas de checkpoint entre chaque — la voix est calée, l'enchaînement compte plus que l'arrêt.
- Pour chaque section : suivre **Promesse + Points clés + Artefacts** du plan. Ne pas inventer de nouvelle section non prévue ; si une section du plan se révèle vide en cours de route, le signaler à la fin plutôt que de boucher au mortier.
- **Code et chemins** : citer les fichiers avec leur chemin réel. Inclure du code court, autosuffisant, commenté inline. Préférer ` ```ts` / ` ```php` / ` ```bash` etc. avec le langage explicite — c'est ce qui donne la coloration syntaxique.
- **Liens** : utiliser des liens markdown classiques `[texte](url)`. La traduction des liens internes inter-langues se fait en étape 6 si pertinent.
- **Citations** : les sources externes citées dans `## Synthèse de recherche` du plan apparaissent dans le corps via lien inline. Ne pas inventer d'URL ; si une source est marquée *« à étayer »*, garder un placeholder visible (`[source à confirmer]`) plutôt qu'une URL inventée.
- **Composants MDX éventuels** : si la collection détectée utilise des composants (Callout, Aside, Figure...), les utiliser quand ça améliore la lecture, mais ne pas en inventer un nouveau. S'aligner sur ce que les articles existants emploient.
- **Garde-fous du plan** : la section `## Risques & garde-fous` du plan est ta checklist anti-dérive. Relire avant chaque section sensible.

### 4. FAQ + frontmatter final

Une fois le corps rédigé :

- **FAQ** : si le schéma a un champ `faq` et que le plan liste 2–4 questions, rédiger les réponses maintenant. Réponses courtes (2–4 phrases), qui ajoutent ou nuancent — pas un copier-coller de l'article. Si le plan ne liste pas de FAQ et que le schéma le supporte, demander au user s'il en veut.
- **Frontmatter** : remplir tous les champs **en respectant le schéma détecté en étape 0**. Pour chaque champ avec borne explicite (Astro Zod : `min`, `max`, `enum`), **compter** ou **vérifier** — ne pas se fier à l'estimation. Pour les stacks à frontmatter libre, calquer la structure des articles existants.

Points de vigilance fréquents :

- Bornes de longueur sur les champs résumés (excerpt, description, tldr) — souvent étroites, faciles à dépasser ou rater.
- Enums fermés (catégorie, statut) — utiliser exactement une valeur du jeu existant, ne pas en créer de nouvelle.
- Compteurs partagés entre versions linguistiques — relire la valeur du plan, vérifier qu'aucun article publié depuis n'a pris ce numéro.
- `lang` / `locale` selon convention détectée.
- `translationOf` (ou équivalent) : laisser **vide** tant que la traduction n'a pas été produite — sera ajouté en étape 6 sur les **deux** versions de manière réciproque.
- Champ `draft` selon réponse de l'étape 1.

### 5. Vérifications mécaniques

Avant de rendre la main, lancer **dans cet ordre** les commandes détectées en pré-requis. Corriger les erreurs avant de passer à l'étape suivante :

1. **Vérification schéma / typecheck** (ex. `npm run check`, `astro check`, `next build`, `tsc --noEmit`). C'est la barrière schéma. Toute erreur ici = frontmatter cassé ou MDX mal formé. Réparer (souvent : longueur d'un champ hors bornes, enum mal cassée, date mal formatée).
2. **Lint** (ex. `npm run lint`, ou `npm run lint:fix` si auto-réparation possible). Utile si l'article contient des composants MDX.
3. **Format** (ex. `npm run format`, `prettier --write`). Vérifier que le frontmatter YAML reste valide après formatage (certains formatters reflowent le YAML, attention aux multilignes).

Lancer en parallèle quand c'est possible, séquentiel si une commande dépend de la précédente.

Si une commande échoue après réparation : ne pas l'ignorer, ne pas faire passer en force. Remonter le diagnostic au user et arbitrer ensemble.

Si **aucune commande de vérification n'a été détectée** (cas markdown brut sans tooling) : faire au minimum une relecture du frontmatter à l'œil pour vérifier qu'il parse en YAML, et que la structure de fichier match les articles existants.

**Optionnel** : si le user veut voir le rendu avant de figer, proposer de lancer le serveur de dev du projet (`npm run dev`, `hugo serve`, `bundle exec jekyll serve`...) puis ouvrir l'URL de l'article. Utile sur les articles avec beaucoup de code ou de listes — le rendu visuel attrape parfois ce que l'œil rate dans le source.

### 6. Traduction — si et seulement si prévue dans le plan

Lire `## Frontmatter prévisionnel` → ligne **Pendant traduit**. Quatre cas :

- **Sans objet (monolingue)** → s'arrêter ici. Rendre la main.
- **Pas prévu** → s'arrêter ici.
- **À décider après source validée** → demander au user maintenant : *« La version source est figée. On enchaîne la traduction ou on s'arrête ici ? »*. S'il dit non, s'arrêter.
- **Prévu** → enchaîner la traduction.

**Si on enchaîne :**

1. **Slug traduit** : utiliser celui anticipé dans le plan. Si absent du plan, le proposer au user avant de créer le fichier. Convention par défaut : traduire le slug, ne pas suffixer `-en`/`-fr`.
2. **Créer le fichier** dans la collection cible avec l'extension détectée (`.mdx` / `.md`). Respecter la convention de nommage de la stack — Jekyll par exemple impose `<date>-<slug>.md` dans `_posts/`, Hugo accepte `<slug>/index.md` ou `<slug>.md`.
3. **Traduction, pas réécriture.** La voix, le ton, le rythme, les exemples, les liens externes restent. Adapter idiomatiquement, ne pas calquer mot à mot. Les noms de fichiers, classes, variables techniques ne se traduisent pas.
4. **Liens internes** : si l'article source pointe vers un autre article du blog, basculer vers la version traduite si elle existe. Sinon, garder le lien source — mieux qu'un lien cassé.
5. **Frontmatter traduit** :
   - `lang` / `locale` : code de la langue cible.
   - Pointeur de traduction (`translationOf`, `alternate`...) selon convention détectée : pointer vers le slug source **sans extension**.
   - **Champs partagés** (ex. `number` dans Astro CC) : reprendre la même valeur que la version source. C'est la séquence éditoriale, partagée.
   - `category`, `tags`, `keywords` : traduits si pertinent (les noms de techno restent).
   - Champs résumés (`title`, `excerpt`, `tldr`...) : traduits, **revérifier les bornes** — la traduction peut faire dériver la longueur.
   - `faq` : questions **et** réponses traduites.
6. **Réciprocité** : ajouter le pointeur de traduction **rétroactivement sur la version source** vers le slug traduit. Sans cette réciprocité, les balises `hreflang` (ou équivalent) sont incomplètes côté SEO.
7. Re-lancer **étape 5** (vérifications) sur le repo entier — la version traduite doit valider au même titre que la source.

### 7. Bilan & prochaine étape

Rendre la main avec un récapitulatif court :

- Fichier(s) créé(s) avec chemin complet.
- Statut `draft` (true / false / sans objet).
- Vérifications passées (oui / non, et lesquelles).
- Reste à faire : choix encore ouverts, sources à étayer, captures à ajouter, OG image à générer, mise en ligne, etc.
- Si pertinent : suggérer de lancer le dev server pour relire le rendu, ou (après merge) `/workflow:review` sur le diff pour audit final.

Pas de commit automatique. Le user décide quand commiter et quoi inclure.

## Conventions et garde-fous du skill

- **Le plan est la loi.** Si le user demande en cours de rédaction *« ajoute une section sur X »* alors que X n'est pas dans le chapitrage : faire l'aller-retour explicite — *« Cette section n'est pas dans le plan. On l'ajoute (et on met à jour le plan), ou on la garde pour un futur article ? »*. Ne pas étendre silencieusement.
- **Pas de génération hors plan.** Pas de meta-conclusion ajoutée si le plan n'en prévoit pas (souvent la dernière section porte la chute — c'est explicite dans `article-plan`). Pas de « Ressources » / « Pour aller plus loin » / « À propos de l'auteur » hors plan.
- **Pas d'invention de chiffres ni d'URLs.** Une donnée non sourcée → marquer `[à étayer]` visible ; une URL non vérifiée → ne pas la mettre. Préférer un trou visible à un faux comblé.
- **Respect strict du schéma au moment d'écrire le frontmatter**, pas en fin de build. Compter les caractères, ne pas estimer. Vérifier les enums.
- **Voix calquée sur les articles de référence**, sauf indication contraire du plan. Pas de bascule de voix en cours d'article.
- **Tics à éviter** : em-dashes en pagaille (un par paragraphe max), emojis, MAJUSCULES emphatiques, *« dans cet article nous verrons »*, relances LinkedIn-like (« vous êtes prêt ? »).
- **Slug = nom de fichier**, sans accent, kebab-case, identique à celui du plan sauf si le user demande explicitement de le changer (auquel cas : renommer le dossier `docs/story/a-NNN-slug` aussi, par cohérence).
- **Compteurs partagés entre versions linguistiques** (si la stack en a) : vérifier qu'aucun article publié entre l'écriture du plan et la rédaction n'a pris la valeur prévue.
- **Le checkpoint après section 1 ne se saute pas.** C'est le seul vrai contrôle qualité humain du skill. Tenir bon même sous pression.
- **Les commandes de vérification doivent passer avant de rendre la main.** Une rédaction qui casse le build n'est pas terminée. S'il n'y a pas de commande de vérification (markdown brut), faire au minimum une relecture frontmatter à l'œil.

## Exemples d'amorçage

**Exemple 1 — exécution directe sur stack typée :**

User : *« Rédige l'article depuis `docs/story/a-001-quitter-hexagonale-pour-adr-symfony/plan.md`. »*

→ Lire le plan. Re-détecter stack (Astro CC). Lire `src/content.config.ts` pour le schéma à jour. Lire les articles de référence cités. Vérifier que `quitter-hexagonale-pour-adr-symfony.mdx` n'existe pas déjà. Cadrage rapide (3 questions : draft, date, choix encore ouverts — par exemple choisir le titre parmi 3 variantes). Écrire intro + section 1. Checkpoint. Continuer. FAQ. Frontmatter. `npm run check` + `lint` + `format`. Demander la traduction si le plan dit *« décision après source validée »*.

**Exemple 2 — plan incomplet :**

User : *« Rédige a-002 »*. Le plan a une thèse molle (« comprendre les RAG ») et pas de chapitrage.

→ Refuser de rédiger, expliquer ce qui manque, proposer de relancer `article-plan` sur ce slug pour itérer. Un plan creux ne se rattrape pas en rédaction.

**Exemple 3 — reprise après checkpoint :**

User : *« Continue, mais resserre l'intro, elle est trop longue, coupe le 2e paragraphe. »*

→ Appliquer la coupe, montrer la nouvelle intro + section 1, redemander un OK explicite **avant** d'enchaîner sur la section 2. Ne pas filer dans la rédaction tant que la voix n'est pas validée.

**Exemple 4 — side-project sur stack typée :**

User : *« Rédige la fiche side-project depuis `docs/story/a-005-otel-tracing-cli/plan.md`. »*

→ Mode project. Cible la collection projects détectée (chemin + extension). Schéma project (souvent plus narratif, pas de `tldr`/`number`). Reste du workflow identique. Souvent plus court (800–1500 mots), ton plus orienté narration produit.

**Exemple 5 — markdown brut sans tooling :**

User : *« Rédige depuis a-007-feature-flags. »* — projet en markdown pur, dossier `articles/`, pas de schéma typé, pas de `package.json`.

→ Détecter : stack `markdown brut`, collection `articles/`, extension `.md`, frontmatter calqué sur les articles existants (ex : `title`, `date`, `tags`). Pas de commande de vérification → relecture frontmatter à l'œil en étape 5. Le reste du workflow tourne identiquement.

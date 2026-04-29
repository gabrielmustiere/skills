---
name: article-rework
description: Retouche chirurgicale d'une portion d'un article déjà publié — un chapitre, une section, un paragraphe — sans rejouer la rédaction complète. Resserrer, étoffer, changer le ton, restructurer, corriger un angle qui dévie de la thèse. Lit l'article cible dans la collection détectée (Astro CC, Next.js MDX, Hugo, Jekyll, markdown brut), lit le `plan.md` associé sous `docs/story/a-<NNN>-<slug>/`, applique la retouche en respectant la voix de l'article, met à jour le plan si la promesse de la section change, propage à la traduction si une version traduite existe. Déclenche sur "retravaille cette section", "réécris le chapitre X", "ce paragraphe est trop long resserre-le", "étoffe la section sur Y", "le ton dérape ici", "cette partie ne sert plus la thèse", "refonds le passage sur Z", "raccourcis l'intro de cet article", "remplace cet exemple". À utiliser dès que le user pointe une portion d'un article publié à modifier, pas à utiliser pour une rédaction from scratch (`article`) ni pour un cadrage initial (`article-plan`).
metadata:
  version: 0.1.0
---

# Article Rework

Retouche chirurgicale d'une **portion** d'un article déjà rédigé : un chapitre, une section, un paragraphe ciblé. Pas une réécriture complète, pas un nouveau cadrage — une intervention locale qui respecte la voix, la thèse et la structure du reste de l'article.

Entrée : un article existant dans la collection détectée + une portion désignée par le user.
Sortie : la même portion réécrite à l'endroit, le frontmatter mis à jour si nécessaire (compteur de mots, excerpt, date de mise à jour), le plan synchronisé si la promesse de la section change, la traduction propagée si une version existe, vérifications passées.

## Pourquoi ce skill existe

Retravailler une section d'un article publié paraît trivial — c'est un piège. Trois risques propres à cet exercice justifient un skill dédié :

1. **La voix qui se brise.** Un article publié a sa voix calée sur l'ensemble du texte. Une réécriture isolée a tendance à parler une voix différente — celle du moment, sans le contexte des 3000 mots autour. Le résultat : un patch lisible en isolation, mais qui détonne à la lecture continue. → Avant d'écrire, **relire l'article complet**, repérer les tics récurrents (rythme, longueur de phrase, voix je/on, niveau de jargon, transitions), et calquer.
2. **La thèse qui dérive.** Chaque section sert une promesse précise (formulée dans le `plan.md`). Une retouche qui étoffe ou recadre la section peut, sans qu'on s'en rende compte, déplacer la thèse de l'article. Si la nouvelle version de la section ne sert plus la thèse globale, l'article devient incohérent. → **Lire le plan avant de réécrire**, traiter la **Promesse + Points clés** de la section comme un contrat. Si la retouche le viole, c'est un signal — soit on renonce, soit on met à jour le plan en conscience.
3. **Le frontmatter et la traduction qui se désynchronisent.** Modifier une section change souvent la longueur du corps (impact sur `wordCount` / `readingTime`), peut invalider l'`excerpt` (qui résumait l'ancienne version), et désaligne la version traduite si elle existe. → **Vérifier le frontmatter en fin de retouche**, propager à la traduction dans le même appel si elle existe.

Tout le reste — syntaxe, code, liens — découle d'une lecture attentive de la portion dans son contexte.

## Pré-requis : ne jamais réécrire à l'aveugle

Avant la moindre proposition, **toujours** :

1. **Lire l'article complet**, pas juste la portion ciblée. La voix vit dans l'ensemble. Repérer en passant : longueur moyenne des phrases, listes vs prose, ratio code/texte, voix (je / on / impersonnel), tics récurrents, conventions de titres, transitions entre sections.
2. **Lire le `plan.md`** associé s'il existe (`docs/story/a-<NNN>-<slug>/plan.md`). En particulier : `## Thèse`, `## Chapitrage` (la **Promesse** + les **Points clés** de la section visée), `## Tonalité`, `## Risques & garde-fous`. Si le plan n'existe pas (article importé, antérieur au workflow), le signaler — la retouche se fait alors à partir de la voix du seul article, sans contrat de promesse explicite.
3. **Re-détecter la stack et les commandes** (Astro CC, Next MDX, contentlayer, Hugo, Jekyll, markdown brut) — mêmes critères qu'`article` étape 0. La stack a pu bouger depuis la rédaction initiale. Identifier en particulier les commandes de **schéma / typecheck**, **lint**, **format**.
4. **Lire le ou les articles de référence** cités dans `## Tonalité` du plan si présent. Pas obligatoire si le plan n'existe pas — l'article cible lui-même fait office de référence dans ce cas.
5. **Si l'article a une version traduite** (champ `translationOf` ou équivalent dans le frontmatter), localiser le fichier traduit dès maintenant. Pas pour le lire en détail tout de suite, juste pour savoir qu'il existe et qu'il faudra le toucher en fin de cycle.

## Workflow

### 0. Identifier l'article et la portion

Le user pointe presque toujours une cible claire (« retravaille la section *Pourquoi pas mock* dans `quitter-hexagonale-pour-adr-symfony` »), mais pas toujours.

1. **Identifier le fichier source** dans la collection détectée. Si plusieurs candidats (FR/EN, slug ambigu), demander.
2. **Identifier la portion exacte** :
   - Section entière (titre `##` ou `###` désigné).
   - Sous-bloc précis (paragraphe, exemple de code, liste).
   - L'intro (avant le premier `##`).
   - La FAQ (un item ou plusieurs).
   Si le user donne un nom de section qui n'existe pas tel quel, **ne pas inventer** — lister les sections existantes et demander laquelle.
3. **Identifier le `plan.md`** associé via le slug. Si le mapping n'est pas trivial (slug renommé, plan archivé), demander confirmation au user plutôt que de deviner.

### 1. Cadrer la nature de la retouche — 1 question, pas plus

Le user a déjà dit ce qu'il veut, mais l'intention peut être ambiguë. Reformuler en une phrase ce qu'on a compris :

> *« OK, donc tu veux **resserrer** la section *X* (actuellement ~Y mots, on vise quoi — moitié, deux tiers ?). On garde la promesse *« <Promesse extraite du plan> »*, on coupe juste les redondances et l'exemple Z. C'est bien ça ? »*

Reformulation utile sur trois axes :

- **Type d'intervention** : resserrer / étoffer / changer le ton / restructurer / corriger l'angle / remplacer un exemple. Demander seulement si le brief est ambigu — ne pas faire un tour de questions inutile.
- **Périmètre** : strictement la portion désignée, ou autorisé à toucher les phrases de transition autour (souvent nécessaire pour que la retouche ne se voie pas) ?
- **Contrainte explicite** : longueur cible, ton spécifique, exemple à intégrer, élément à supprimer.

Une seule question groupée, pas un interrogatoire. Si le brief est déjà précis (« coupe le 2e paragraphe, c'est de la redite »), pas de question — exécuter.

### 2. Vérifier la cohérence avec le plan

Avant d'écrire la nouvelle version, comparer la retouche demandée à la **Promesse** + **Points clés** de la section dans le plan :

- **La retouche reste dans la promesse** → exécuter normalement.
- **La retouche change la promesse** (le user veut que la section démontre autre chose) → c'est un signal fort. Trois options à proposer :
  1. Retouche dans le périmètre actuel (on garde la promesse du plan).
  2. Retouche **et** mise à jour du plan pour refléter la nouvelle promesse — à valider explicitement par le user.
  3. La retouche est en réalité un **nouvel article** ou un retour à `article-plan` pour repenser la structure.
- **La retouche fait dériver la thèse globale** (la section n'est plus alignée avec `## Thèse` du plan) → ne pas exécuter en silence. Remonter le constat, demander si la thèse de l'article doit évoluer (auquel cas on touche au plan), ou si la retouche doit être ajustée.

Si **aucun plan n'existe**, sauter cette étape mais le signaler une fois : *« Pas de `plan.md` trouvé pour cet article — je travaille à partir de la voix du seul article, sans contrat de promesse. Si la retouche change ce que démontre la section, je n'ai pas de filet. »*.

### 3. Proposer la nouvelle version → CHECKPOINT

Écrire la nouvelle version de la portion, en respectant :

- **La voix de l'article**, pas une voix générique. Calque sur le rythme, le niveau de jargon, les tics observés à l'étape pré-requis. Si l'article emploie « on », ne pas basculer en « nous ». Si les phrases sont courtes et incisives, ne pas pondre des paragraphes denses.
- **Les conventions du fichier** : composants MDX déjà utilisés dans l'article (Callout, Aside...), conventions de blocs de code (langage explicité, chemin du fichier en commentaire), façon dont les liens sont placés.
- **La promesse de la section** (si plan présent), même si la formulation change.
- **Les transitions** : si la retouche modifie la fin de la section, vérifier que le début de la section suivante reste cohérent (parfois une phrase de raccord à ajuster).

Présenter au user **avant d'écrire dans le fichier** :

- **Avant** (l'extrait actuel, tel quel).
- **Après** (la version proposée).
- Une note brève : ce qui a changé, ce qui a été préservé, et **un drapeau visible** si quelque chose mérite arbitrage (« j'ai supprimé l'exemple Foo, j'ai gardé Bar ; tu veux qu'on remette Foo plus court ? »).

Puis **arrêter** et attendre validation. Message explicite :

> *« Voici la proposition. Je n'écris rien dans le fichier tant que tu n'as pas dit OK ou pointé ce qu'il faut ajuster. »*

**Ne pas continuer** sans feedback explicite. Si le user demande une modif ciblée, l'appliquer **puis re-checkpoint** — ne pas filer écrire le fichier dans la foulée d'un nouveau jet.

C'est le contrôle qualité principal du skill. Ce checkpoint ne se saute pas, même si le user dit *« vas-y direct »* — expliquer brièvement pourquoi (une retouche écrite sans relecture cassse plus souvent la voix de l'article qu'elle ne la corrige) et reproposer le checkpoint.

### 4. Appliquer la retouche dans le fichier

Une fois validée, écrire au bon endroit dans le fichier source. Préférer un `Edit` ciblé (remplacement exact de la portion) plutôt qu'un `Write` du fichier complet — plus sûr, plus lisible dans le diff, ne touche pas par accident le reste.

Vérifier dans la foulée :

- **Le texte autour** : la phrase qui précède et celle qui suit la portion modifiée s'enchaînent naturellement. Si la retouche a supprimé une référence (« comme on l'a vu plus haut... »), vérifier que cette référence pointe encore vers quelque chose qui existe.
- **Les ancres internes** : si l'article contient des liens vers une ancre (`#mon-titre`) et que la retouche a renommé un titre, vérifier que les ancres tiennent toujours.
- **La FAQ** : si la portion modifiée invalide une réponse de FAQ (la FAQ disait « la section X explique Y » et X n'explique plus Y), mettre à jour la FAQ.

### 5. Mettre à jour le frontmatter si pertinent

Une retouche modifie souvent — pas toujours — des champs du frontmatter :

- **`wordCount` / `readingTime`** : si le schéma a ces champs, recalculer.
- **`excerpt` / `description` / `tldr`** : si l'extrait de tête résumait l'ancienne version d'un point qui a changé, le réajuster. **Attention aux bornes** (Astro Zod : `min`/`max`) — recompter, ne pas estimer.
- **`updatedAt`** : si le schéma a ce champ, le passer à la date du jour.
- **`tags` / `keywords`** : si la retouche introduit ou supprime un thème majeur, ajuster — mais pas pour un mot ajouté ou retiré.
- **Ne pas toucher** au compteur partagé (`number` Astro CC), à `lang`, à `translationOf`, à la date initiale de publication.

Si aucun de ces champs n'a besoin de bouger, ne pas y toucher (les diffs propres comptent).

### 6. Mettre à jour le plan si la retouche le justifie

À cette étape, **si l'étape 2 a identifié que la promesse de la section change** (et que le user a validé), mettre à jour `docs/story/a-<NNN>-<slug>/plan.md` :

- Ajuster la **Promesse** + **Points clés** de la section concernée dans `## Chapitrage`.
- Si la thèse globale de l'article a évolué, mettre à jour `## Thèse` également.
- Ajouter une ligne datée dans une section `## Historique` ou `## Notes` si la convention du plan le prévoit (à inférer en regardant le plan existant).

Ne pas modifier le plan si la retouche reste strictement dans la promesse. Le plan n'est pas un journal de modifs — c'est le contrat éditorial.

Si **aucun plan n'existe**, ne rien créer ici — `article-plan` est le skill qui crée des plans, pas celui-ci.

### 7. Propager à la traduction si elle existe

Si l'article a une version traduite (repérée à l'étape pré-requis) :

1. **Localiser dans le fichier traduit la portion équivalente** : par titre traduit, par position dans le chapitrage, ou en lisant les deux fichiers en parallèle.
2. **Reformuler la même retouche en langue cible**, pas un calque mot à mot. Idiomatique, pas littéral. Garder noms de fichiers / classes / variables techniques inchangés.
3. **Vérifier le frontmatter du fichier traduit** : mêmes ajustements qu'à l'étape 5 (longueurs des champs résumés, `updatedAt`, etc.). Recompter les bornes — la traduction a souvent une longueur différente.
4. **FAQ traduite** : si la FAQ source a été touchée, traduire le changement aussi.
5. **Ancres internes** : si l'article traduit utilise les mêmes ancres ou des ancres traduites, ajuster en conséquence.

Présenter le diff de la traduction au user de la même manière qu'à l'étape 3 (avant / après), même checkpoint. La voix de la version traduite est **sa voix**, pas un décalque automatique de la version source — la valider explicitement.

### 8. Vérifications mécaniques

Avant de rendre la main, lancer **dans cet ordre** les commandes détectées au pré-requis. Corriger les erreurs avant de passer à la suivante :

1. **Vérification schéma / typecheck** (`astro check`, `next build`, `tsc --noEmit`, `hugo --check`...). Une retouche peut faire dériver un champ résumé hors bornes — c'est ici qu'on s'en rend compte.
2. **Lint** (`npm run lint`, `markdownlint`...). Utile si la retouche a introduit / modifié des composants MDX.
3. **Format** (`npm run format`, `prettier --write`).

Lancer en parallèle quand possible. Si une commande échoue : pas de passage en force. Diagnostic remonté, arbitrage avec le user.

Si **aucune commande de vérification n'a été détectée** (markdown brut sans tooling) : relire le frontmatter à l'œil, vérifier que le YAML parse, que la structure de fichier reste cohérente avec les autres articles.

**Optionnel** : proposer de lancer le serveur de dev pour visualiser le rendu de la section retouchée. Particulièrement utile si la retouche concerne du code, des listes, ou des composants MDX.

### 9. Bilan & prochaine étape

Rendre la main avec un récapitulatif court :

- Fichier(s) modifié(s) avec chemin complet.
- Portion retouchée (titre de section ou repère).
- Type d'intervention (resserrer / étoffer / restructurer...).
- Plan mis à jour ? oui / non / sans objet.
- Traduction propagée ? oui / non / sans objet.
- Vérifications passées ? oui / non, lesquelles.
- Reste à faire : sources à confirmer, captures à régénérer si la retouche en invalide une, mise à jour de l'OG image si le titre a changé, etc.
- Si pertinent : suggérer `/workflow:review` sur le diff pour audit final.

Pas de commit automatique. Le user décide quand commiter et quoi inclure.

## Conventions et garde-fous du skill

- **Le périmètre est la portion désignée.** Si en lisant l'article on repère un autre passage qui mériterait une retouche, **le signaler à la fin** (« j'ai vu que la section Z a un exemple obsolète — on l'attaque maintenant ou plus tard ? »), ne pas étendre silencieusement.
- **La voix de l'article prime sur la voix générique.** Calquer sur le texte cible, pas sur des patterns par défaut.
- **Le plan est le contrat.** Si la retouche viole la promesse de la section, c'est une décision explicite (mise à jour du plan + validation) — pas un fait accompli.
- **Pas de réécriture intégrale déguisée.** Si plus de ~70% de la portion change, la retouche est en réalité un nouveau jet — proposer au user de revenir à `article` pour la section concernée plutôt que de prétendre à une retouche.
- **Pas d'invention de chiffres ni d'URLs.** Une donnée ajoutée doit être sourçable (le user fournit, ou marquer `[à étayer]` visible).
- **Le checkpoint avant écriture ne se saute pas.** C'est le seul vrai contrôle qualité humain — l'enjamber transforme le skill en générateur de patches qui sonnent faux.
- **Le frontmatter se vérifie après.** Une retouche silencieuse qui pète une borne `excerpt` casse le build au prochain merge.
- **La traduction est sa propre voix.** Pas un Google Translate du diff français — un re-jet idiomatique de la même retouche en langue cible, soumis au même checkpoint.
- **Les commandes de vérification doivent passer avant de rendre la main.** Pareil que `article` : casser le build n'est pas terminer.

## Exemples d'amorçage

**Exemple 1 — resserrer une section trop longue :**

User : *« Dans `quitter-hexagonale-pour-adr-symfony`, la section *Pourquoi pas mock* fait 600 mots, c'est trop. Coupe à ~300, garde l'argument central, vire les deux exemples redondants. »*

→ Lire l'article complet, lire le plan (la promesse de cette section : « expliquer pourquoi on a renoncé aux mocks »), lire les articles de référence cités. Identifier la section. Proposer une version à 300 mots qui garde l'argument central (premier exemple) et coupe les deux exemples redondants. Présenter avant/après. Checkpoint. Appliquer. Vérifier `wordCount` global du frontmatter (a baissé, mettre à jour). Pas de plan à modifier (promesse intacte). Article monolingue → pas de traduction. `npm run check` + `lint` + `format`. Bilan.

**Exemple 2 — étoffer une section pauvre :**

User : *« La section *Migration progressive* est creuse, étoffe avec un exemple concret. »*

→ Lire le plan : promesse = « montrer comment on a migré sans big bang », points clés = « stratégie strangler, feature flag, métriques de bascule ». Constat : la section actuelle ne couvre que la stratégie strangler. Demander : *« Le plan prévoyait aussi feature flag + métriques — on étoffe sur ces deux axes manquants, ou tu veux un autre exemple précis ? »*. User répond : *« feature flag, le user a une expérience concrète sur ce sujet »*. Demander un détail concret au user (quelle techno, quel projet). Rédiger. Checkpoint. Appliquer. `wordCount` recalculé. Vérifications. Si l'article a une version EN avec `translationOf`, propager dans la foulée.

**Exemple 3 — la retouche change la promesse de la section :**

User : *« La section *Quand utiliser ADR* devrait plutôt parler de *Comment écrire un bon ADR*, recadre. »*

→ Lire le plan. Constat : la promesse actuelle est *« savoir reconnaître les cas qui méritent un ADR »*, le user veut basculer vers *« écrire un ADR utile »*. Ce n'est pas une retouche, c'est un changement de promesse. Remonter au user les trois options : (1) garder la promesse actuelle, (2) la changer + mettre à jour le plan, (3) c'est un nouvel article. User répond (2). Mettre à jour `## Chapitrage` du plan. Vérifier que `## Thèse` tient toujours (a priori oui, juste un point clé déplacé). Rédiger la nouvelle section. Checkpoint. Appliquer. Plan mis à jour. Traduction propagée. Vérifications. Bilan signale clairement que le plan a évolué.

**Exemple 4 — pas de plan associé :**

User : *« Retravaille l'intro de `mon-vieux-article-2023.mdx`, elle ouvre par une définition, c'est mou. »*

→ Article importé, pas de `docs/story/a-*` correspondant. Signaler une fois : *« Pas de plan trouvé, je travaille à partir de la voix du seul article. »*. Lire l'article complet pour caler la voix. Reformuler l'intro en partant d'une situation concrète (pas une définition), tout en gardant la thèse implicite de l'article tel qu'il est. Checkpoint. Appliquer. `excerpt` invalide ? recalculer. Vérifications.

**Exemple 5 — restructurer (ordre des idées) :**

User : *« Dans la section *Mise en place du retry*, l'exemple de code arrive avant l'explication, c'est l'inverse de tout le reste de l'article — réordonne. »*

→ Lire l'article complet, confirmer la convention (explication → code), confirmer en lisant le plan que la promesse n'est pas affectée. Réordonner : déplacer l'explication avant le bloc de code, réécrire la phrase de raccord. Aucun mot supprimé ni ajouté significatif → pas d'impact `wordCount`. Checkpoint. Appliquer. Vérifications. Pas de plan à modifier (juste un ordre interne).

---
name: styles
description: Customise les styles Sylius 2.x (admin Tabler ou shop Bootstrap) : surcharge les variables CSS (`--tblr-*`, `--bs-*`) via SCSS sans patcher vendor. `assets/<ctx>/styles/custom.scss`, import entrypoint.js, `yarn build`.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /styles — Customiser les styles Sylius

Tu aides à **modifier l'apparence visuelle** de l'admin ou du shop Sylius 2.x. Le pattern officiellement recommandé : **surcharger les variables CSS** des frameworks frontend (Tabler UI côté admin, Bootstrap côté shop) dans un fichier SCSS applicatif, importé via l'`entrypoint.js`, puis recompilé avec `yarn`. Aucun patch du vendor, aucune duplication de feuille de style.

Référence officielle : [docs.sylius.com/the-customization-guide/customizing-styles](https://docs.sylius.com/the-customization-guide/customizing-styles). Variables : [Tabler](https://tabler.io/) (admin), [Bootstrap](https://getbootstrap.com/docs/5.3/customize/css-variables/) (shop).

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine. Vérifier `sylius/sylius` ≥ 2.0 (le pattern variables CSS est spécifique à Sylius 2.x — sur 1.x, le shop est en Semantic UI et l'approche diffère).
   - Présent → OK.
   - Absent → *« Ce skill cible Sylius 2.x (Tabler/Bootstrap variables). Je ne trouve pas `sylius/sylius`. On continue quand même ? »*
2. Lire `package.json` pour confirmer la présence de `yarn` (ou `npm`) et de la chaîne de build (`@symfony/webpack-encore` ou AssetMapper). Sylius 2.x utilise Encore par défaut.
3. Vérifier l'existence des entrypoints :
   ```bash
   ls assets/admin/entrypoint.js assets/shop/entrypoint.js
   ```
   Absents → c'est une install custom ou un fork ; demander où sont les entrypoints assets avant de toucher.
4. Si la demande est *« changer le layout / la structure HTML »* → c'est un cas **`/sylius:template`** (Twig Hooks), pas du style. Ce skill ne touche que CSS/SCSS.
5. Si la demande est *« thème différent par canal »* → cumuler avec **Sylius Themes** (cf. §6 de `/sylius:template`). Les variables CSS peuvent vivre dans un thème.

## Règles fondamentales

- **Variables CSS d'abord, toujours.** Override les `--tblr-*` (admin) et `--bs-*` (shop) plutôt que de réécrire des règles `.btn-primary { background: ... }`. Une variable change tout l'écosystème de composants qui s'en sert — une règle CSS ne change qu'un endroit et casse à la prochaine refonte du composant.
- **Ne jamais éditer le vendor.** `vendor/sylius/.../assets/...` saute au prochain `composer update`. Tout passe par `assets/admin/` ou `assets/shop/` du projet.
- **`yarn build` après chaque modif SCSS.** Sans rebuild, le fichier `public/build/` reste sur l'ancienne version compilée. En dev, `yarn watch` rebuild en continu.
- **`cache:clear` après le build.** Symfony cache les manifests d'assets ; sans purge, le navigateur peut servir l'ancien hash. Combiné à un hard refresh navigateur (`Cmd+Shift+R` / `Ctrl+Shift+R`) pour court-circuiter le cache HTTP.
- **Inspecter avant d'écrire.** Toujours ouvrir DevTools sur l'élément cible et lire le panneau **Styles** : la variable exacte (`--tblr-btn-bg`, `--bs-link-color`, etc.) est visible dans les `Computed`/`Styles`. Tirer au hasard une variable depuis la doc Bootstrap/Tabler sans inspecter mène à des heures de debug.
- **Sélecteur ciblé > sélecteur universel.** `.btn-primary { --tblr-btn-bg: ... }` ne touche que les boutons primaires. `* { --tblr-primary: ... }` propage à toute la cascade — utile pour une refonte de marque, dangereux pour une retouche locale.
- **`!important` réservé aux overrides sur `*` / `:root`.** Bootstrap/Tabler définissent leurs variables sur `*` et `:root` avec une spécificité très forte. Sans `!important`, ton override sur `*` peut être ignoré. Mais sur un sélecteur de classe normal, `!important` est à proscrire.

## Déroulement

### 1 — Cadrer le besoin

Demander (ou confirmer) :

- **Cible** : `admin` (Tabler) ou `shop` (Bootstrap) ?
- **Portée** : un composant précis (bouton "Add to cart", lien d'un menu, badge promo) ou la couleur primaire globale du thème ?
- **Direction** : changer une couleur, un radius, une espacement, une typo ? Tabler et Bootstrap exposent ~300 variables chacun — la doc ne sert qu'après inspection.

### 2 — Identifier la variable

Méthode unique fiable : **DevTools → Inspect → Styles**.

1. Ouvrir l'admin (`/admin/...`) ou le shop dans le navigateur en mode `dev` (Symfony en `APP_ENV=dev`).
2. `Cmd+Shift+C` (macOS) / `Ctrl+Shift+C` (Windows/Linux), survoler l'élément cible.
3. Dans le panneau **Styles**, repérer les `--tblr-*` (admin) ou `--bs-*` (shop) appliqués.
4. Suivre la chaîne : une variable peut renvoyer à une autre (`--tblr-btn-bg: var(--tblr-primary)`). Si tu changes `--tblr-primary`, tous les boutons et liens qui en dépendent changent en cascade — c'est souvent ce qu'on veut pour un re-branding.

Cas typiques :

| UI cible | Variable usuelle |
| --- | --- |
| Bouton primaire admin | `--tblr-btn-bg`, `--tblr-btn-hover-bg` |
| Couleur primaire admin (cascade) | `--tblr-primary`, `--tblr-primary-darken` |
| Bouton primaire shop ("Add to cart") | `--bs-btn-bg`, `--bs-btn-hover-bg` |
| Couleur primaire shop (cascade) | `--bs-primary`, `--bs-primary-rgb` |
| Lien shop | `--bs-link-color`, `--bs-link-hover-color`, `--bs-link-color-rgb` |
| Navbar shop | `--bs-navbar-hover-color`, `--bs-navbar-active-color` |

### 3 — Créer / éditer le fichier SCSS

#### Côté admin

```scss
// assets/admin/styles/custom.scss

// Override ciblé : seuls les boutons primaires
.btn-primary {
    --tblr-btn-bg: #FF0000;
    --tblr-btn-hover-bg: #FF5531;
}
```

#### Côté shop

```scss
// assets/shop/styles/custom.scss

.btn-primary {
    --bs-btn-bg: #FF0000;
    --bs-btn-hover-bg: #FF5531;
}
```

### 4 — Importer dans l'entrypoint

Le fichier SCSS n'est compilé que s'il est référencé.

#### Admin

```javascript
// assets/admin/entrypoint.js

import './styles/custom.scss';
```

#### Shop

```javascript
// assets/shop/entrypoint.js

import './styles/custom.scss';
```

Vérifier que la ligne d'import existe — s'il y a déjà des imports SCSS, ajouter à la suite, ne pas remplacer.

### 5 — Build des assets

```bash
yarn build       # one-shot prod-like
# ou
yarn watch       # rebuild auto pendant le développement
```

En cas d'erreur Sass (variable inconnue, syntaxe), Encore log dans le terminal — toujours lire la stack avant de relancer.

### 6 — Purge des caches

```bash
php bin/console cache:clear
```

Puis dans le navigateur : `Cmd+Shift+R` (macOS) / `Ctrl+Shift+R` (Windows/Linux) pour forcer un hard refresh et bypasser le cache HTTP du navigateur.

### 7 — Cas d'usage : re-branding global

Pour propager une nouvelle couleur primaire à *toute* la cascade Tabler ou Bootstrap, cibler les variables racines :

#### Admin global

```scss
// assets/admin/styles/custom.scss

* {
    --tblr-primary: #FF0000;
    --tblr-primary-darken: #FF5531;
}
```

Tabler dérive énormément de couleurs depuis `--tblr-primary` (boutons, liens, focus rings, badges). Une seule paire de variables suffit pour la majorité du re-branding.

#### Shop global

Bootstrap est plus granulaire — plusieurs variables doivent être surchargées en parallèle pour un résultat cohérent :

```scss
// assets/shop/styles/custom.scss

* {
    --bs-btn-bg: #FF0000 !important;              // Tous les boutons
    --bs-btn-hover-bg: #FF5531 !important;        // Hover bouton

    --bs-primary-rgb: 255, 0, 0;                   // RGB fallback (utilisé par les rgba())
    --bs-primary: #FF0000;                         // Couleur primaire

    --bs-link-color-rgb: 255, 0, 0;                // Lien (RGB)
    --bs-link-color: #FF0000 !important;           // Lien (hex)
    --bs-link-hover-color: #FF5531 !important;     // Lien hover
    --bs-link-hover-color-rgb: 255, 85, 49 !important;

    --bs-navbar-hover-color: #FF5531;              // Navbar hover
    --bs-navbar-active-color: #FF5531;             // Navbar actif
}
```

`!important` est nécessaire ici parce qu'on attaque le sélecteur universel `*` — sans lui, les définitions Bootstrap sur `*` ou `:root` peuvent l'emporter par ordre d'inclusion. Sur un sélecteur de classe (`.btn-primary`), pas besoin de `!important`.

### 8 — Cas d'usage : retouche locale d'un composant

Pour limiter l'impact (ex. couleur d'un seul badge custom sans bouger le reste), scoper sur la classe ou un parent applicatif :

```scss
// assets/shop/styles/custom.scss

.app-product-badge {
    --bs-badge-bg: #FF0000;
    --bs-badge-color: #FFF;
}
```

Cette approche ne touche que les éléments `.app-product-badge` — les autres badges Bootstrap restent natifs. Recommandé chaque fois que l'override est ciblé, pour limiter la dette visuelle.

### 9 — Dernier recours : override d'élément direct

Si la variable n'existe pas ou ne couvre pas l'effet voulu (rare sur Tabler/Bootstrap récents) :

```scss
// assets/shop/styles/custom.scss

a:hover {
    color: #FF5531 !important;
}
```

À éviter — ça contourne le système de variables et rend la maintenance pénible. Préférer toujours une variable, quitte à fouiller la doc Bootstrap/Tabler pour la trouver.

### 10 — Vérification

1. **Visuel** : recharger la page (hard refresh), inspecter l'élément cible. Le panneau **Styles** doit afficher la nouvelle valeur de la variable.
2. **Compilation** : pas d'erreur Sass dans le terminal `yarn`.
3. **Bundle** : `ls -la public/build/admin.css public/build/shop.css` → mtime récente.
4. **Cache navigateur** : si rien ne bouge malgré tout, ouvrir DevTools → onglet **Network** → cocher *Disable cache* le temps de la session.

### 11 — Clôture

Afficher :

- **Fichiers créés/modifiés** : `assets/<contexte>/styles/custom.scss`, `assets/<contexte>/entrypoint.js`.
- **Commandes exécutées** : `yarn build`, `php bin/console cache:clear`.
- **Variables surchargées** : table des `--xxx-yyy` modifiés et leur nouvelle valeur.
- **Ce qui reste à faire** : tester sur les autres contextes (admin si modif shop et inversement) pour vérifier qu'on n'a pas régressé par effet de bord d'un sélecteur trop large, valider le contraste accessibilité (WCAG) si on touche à des couleurs primaires.

## Pièges fréquents

- **Modifier une règle CSS au lieu d'une variable** : `.btn-primary { background: red }` au lieu de `--tblr-btn-bg: red`. Ça marche visuellement mais casse les états dérivés (hover, focus, disabled) qui s'attendent à dériver de la variable. Toujours pister la variable.
- **Oublier le `!important` sur `*` / `:root`** : override sur sélecteur universel sans `!important` → le navigateur garde la définition Bootstrap/Tabler. Symptôme : la modif passe en DevTools mais reste barrée.
- **Mettre `!important` partout** : spam d'`!important` sur des sélecteurs de classe → impossibilité future de surcharger pour un cas particulier. Réserver à `*` / `:root`.
- **Ne pas rebuild après modif** : SCSS modifié mais `yarn build` oublié → le `public/build/*.css` reste sur l'ancienne version. Si `yarn watch` est lancé, il rebuild seul.
- **Cache Symfony pas purgé** : `cache:clear` oublié → le manifest d'assets pointe sur l'ancien hash. Le navigateur peut charger les bonnes règles, mais pas tout le temps. Toujours purger après build.
- **Cache navigateur** : refresh classique (`F5`) garde le CSS en cache. Hard refresh (`Cmd+Shift+R`) ou *Disable cache* dans DevTools obligatoire pendant le développement.
- **Mauvais entrypoint** : importer dans `assets/admin/entrypoint.js` une feuille de style du shop (ou inversement) → la règle est compilée mais sur le mauvais bundle. Vérifier le contexte.
- **Variable inexistante (typo)** : `--tblr-btn-background` au lieu de `--tblr-btn-bg`. Sass ne lève pas d'erreur (CSS variables sont dynamiques) ; rien ne change visuellement. Toujours copier-coller depuis DevTools.
- **Override d'une variable parent oubliée** : changer `--bs-primary` sans changer `--bs-primary-rgb` → les composants qui utilisent `rgba(var(--bs-primary-rgb), 0.5)` (transparence) restent sur l'ancienne couleur. Toujours surcharger les paires `*` / `*-rgb` ensemble.
- **Toucher au vendor** : modifier `vendor/sylius/.../_variables.scss` → écrasé au prochain `composer update`. Tout passe par `assets/<contexte>/styles/`.

## Argument optionnel

`/sylius:styles admin primary #FF0000` — surcharge la couleur primaire admin (Tabler `--tblr-primary` + `--tblr-primary-darken`).

`/sylius:styles shop primary #FF0000` — surcharge la couleur primaire shop (cascade Bootstrap complète : btn, link, navbar).

`/sylius:styles shop button .btn-primary --bs-btn-bg=#FF0000` — surcharge ciblée d'un composant shop sans toucher au global.

`/sylius:styles inspect` — rappelle la procédure DevTools (`Cmd+Shift+C`, panneau Styles, repérer `--tblr-*` ou `--bs-*`).

`/sylius:styles` sans argument — demande le contexte (admin/shop), la portée (global/composant), l'élément cible, puis guide pas à pas (inspection → SCSS → entrypoint → build → cache).

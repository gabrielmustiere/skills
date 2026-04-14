---
name: test-scenario
description: Joue un scénario utilisateur en temps réel via Playwright MCP — pilote un vrai navigateur sur le shop ou l'admin pour valider un parcours, vérifier un comportement, reproduire un bug. Déclenche dès que l'utilisateur veut "tester un scénario", "vérifier un parcours", "naviguer comme un user", "reproduire ce qui se passe quand…", ou décrit une séquence d'actions à valider, même sans citer le skill.
user_invocable: true
---

# /test-scenario — Test de scénario via Playwright MCP

Tu es un testeur QA. Tu exécutes un scénario utilisateur décrit en langage naturel en utilisant les outils MCP Playwright pour piloter un navigateur en temps réel.

## Périmètre du skill

Ce skill **pilote un navigateur en live** pour rejouer un parcours utilisateur. Il **n'écrit pas de fichier `.spec.ts`** (pour ça, c'est `/implement` qui génère les tests E2E persistés). Il sert à :

- valider qu'une feature fraîchement implémentée se comporte bien dans un vrai navigateur
- reproduire un bug remonté par un utilisateur
- explorer le comportement de l'app sur un parcours qu'on ne connaît pas

## Argument obligatoire

`/test-scenario <description du scénario>`

Exemples :

```
/test-scenario Se connecter en admin, aller dans le catalogue produits, vérifier qu'il y a au moins un produit
/test-scenario Ajouter un produit au panier depuis la page d'accueil shop et aller jusqu'au checkout
/test-scenario Créer un compte client, se connecter, vérifier le dashboard
```

## Environnement

### Channels et URLs

L'application est multi-channel, chaque channel a son propre hostname :

| Channel       | Code           | Hostname        | Shop URL                       |
|---------------|----------------|-----------------|--------------------------------|
| Channel Alpha | `FASHION_WEB`  | `channel-a.wip` | `https://channel-a.wip/fr_FR/` |
| Channel Beta  | `CHANNEL_BETA` | `channel-b.wip` | `https://channel-b.wip/fr_FR/` |

L'admin est accessible depuis n'importe quel hostname :

| Zone  | URL                              | Notes                    |
|-------|----------------------------------|--------------------------|
| Admin | `https://channel-a.wip/admin/`   | Back-office Sylius       |
| API   | `https://channel-a.wip/api/v2/`  | API Platform (si besoin) |

**Important** : les URLs shop ont un préfixe locale obligatoire (`/fr_FR/`).

### Comptes disponibles (depuis les fixtures)

#### Admin (back-office)

| Champ    | Valeur                         |
|----------|--------------------------------|
| URL      | `https://channel-a.wip/admin/` |
| Username | `admin`                        |
| Password | `admin`                        |

Le formulaire de login admin utilise le champ **username** (pas l'email).

#### Clients (shop)

Tous les clients ont le mot de passe : `password123`.

Le formulaire de login shop utilise le champ **email**.

| Email                        | Prénom  | Nom     | Channel        | Hostname        |
|------------------------------|---------|---------|----------------|-----------------|
| jean.dupont@example.com      | Jean    | Dupont  | `FASHION_WEB`  | `channel-a.wip` |
| marie.martin@example.com     | Marie   | Martin  | `CHANNEL_BETA` | `channel-b.wip` |
| pierre.bernard@example.com   | Pierre  | Bernard | `FASHION_WEB`  | `channel-a.wip` |
| sophie.petit@example.com     | Sophie  | Petit   | `FASHION_WEB`  | `channel-a.wip` |
| lucas.robert@example.com     | Lucas   | Robert  | `CHANNEL_BETA` | `channel-b.wip` |
| camille.moreau@example.com   | Camille | Moreau  | `FASHION_WEB`  | `channel-a.wip` |
| thomas.leroy@example.com     | Thomas  | Leroy   | `FASHION_WEB`  | `channel-a.wip` |
| emma.simon@example.com       | Emma    | Simon   | `CHANNEL_BETA` | `channel-b.wip` |
| hugo.laurent@example.com     | Hugo    | Laurent | `FASHION_WEB`  | `channel-a.wip` |
| lea.michel@example.com       | Léa     | Michel  | `CHANNEL_BETA` | `channel-b.wip` |

**Important** : un client doit se connecter sur le hostname de son channel.

## Règles d'exécution

1. **Utiliser exclusivement les outils MCP Playwright** — `browser_navigate`, `browser_snapshot`, `browser_click`, `browser_fill_form`, `browser_press_key`, `browser_take_screenshot`, etc.
2. **Commencer par `browser_navigate`** vers l'URL de départ appropriée au scénario.
3. **Après chaque action, faire un `browser_snapshot`** pour vérifier l'état de la page et déterminer la prochaine action.
4. **Ne pas deviner les sélecteurs** — toujours se baser sur le snapshot pour identifier les éléments interactifs (utiliser les `ref` fournis par le snapshot).
5. **En cas d'erreur ou de page inattendue**, faire un screenshot (`browser_take_screenshot`) et remonter le problème à l'utilisateur.
6. **Ne jamais coder de test Playwright** (pas de fichier `.spec.ts`). Le skill pilote le navigateur en temps réel.
7. **Nettoyage obligatoire en fin de scénario** : supprimer les fichiers temporaires créés sous `.playwright-mcp/` (screenshots, traces) — cf. CLAUDE.md projet.

## Déroulement

### Phase 1 — Analyse du scénario

Décompose le scénario en étapes atomiques. Présente le plan :

```
## Scénario : [description courte]

Étapes prévues :
1. Naviguer vers [URL]
2. [Action]
3. Vérifier [attendu]
...
```

### Phase 2 — Exécution pas à pas

Exécute chaque étape avec les outils Playwright. Après chaque étape, reporte brièvement le résultat :

```
- [x] Étape 1 : Navigation vers /admin/ — OK, page de login affichée
- [x] Étape 2 : Saisie des identifiants — OK
- [ ] Étape 3 : ...
```

Si une étape échoue :

- Capture un screenshot
- Décris ce qui est visible vs ce qui était attendu
- Demande à l'utilisateur s'il faut continuer, adapter le scénario, ou arrêter

### Phase 3 — Bilan

```
## Résultat

- Étapes réussies : X / Y
- Statut : PASS / FAIL
- [Captures d'écran si pertinent]
- [Problèmes rencontrés]
```

### Phase 4 — Nettoyage

- Supprimer les screenshots et traces sous `.playwright-mcp/` qui ne sont plus utiles
- Si le scénario validait une feature implémentée, suggérer la prochaine étape :

> Scénario validé. Prochaine étape : `/review` (si pas encore fait) ou `/commit`.

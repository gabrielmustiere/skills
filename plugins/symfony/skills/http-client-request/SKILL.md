---
name: http-client-request
description: Construit une requête HTTP (symfony/http-client) — HttpClientInterface, scoped_clients, options (json, auth_bearer, timeout). Déclenche sur "appeler API", "HttpClient", "scoped client". Impose un scoped client par API.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /http-client-request — Construire une requête HTTP sortante

Tu construis une requête HTTP sortante (REST, webhook, téléchargement) avec `symfony/http-client`. Tu passes par l'autowiring, tu pousses la config URL/auth/headers communs dans `framework.yaml`, et tu ne laisses dans le code PHP que ce qui dépend de l'appel précis.

## Détection préalable (obligatoire)

1. Lire `composer.json` à la racine du projet.
2. Vérifier `symfony/framework-bundle` et `symfony/http-client`.
   - `symfony/http-client` absent → `composer require symfony/http-client` avant toute chose. Sans ce paquet il n'y a pas d'autowiring ni de configuration `framework.http_client`.
   - `symfony/framework-bundle` absent → stack non-Symfony : demander *« Ce skill cible Symfony, je ne trouve pas `symfony/framework-bundle`. On continue en mode standalone (HttpClient::create) ou on change d'approche ? »* avant d'agir.
3. Vérifier l'extension cURL (`php -m | grep curl`). Absente → le client tombera sur `NativeHttpClient` ; le mentionner (pas de HTTP/2 multiplexé).

## Règles fondamentales

- **Autowiring, toujours** : injecter `HttpClientInterface` dans le constructeur du service. Ne jamais appeler `HttpClient::create()` dans un service — c'est réservé aux scripts standalone et aux tests qui n'ont pas de container.
- **Un scoped client par API externe** dans `framework.yaml` : chaque API (GitHub, Stripe, fournisseur interne) a son entrée sous `http_client.scoped_clients`. `base_uri`, `headers` partagés, `auth_bearer`, `auth_basic`, `timeout` y vivent. Le code PHP ne répète plus ces options.
- **Injection nommée** : pour un scoped client `github.client`, injecter `HttpClientInterface $githubClient` — Symfony fait le matching par nom de paramètre. Si le match échoue, `#[Target('github.client')]`.
- **`json` pour envoyer du JSON**, jamais `body` avec `json_encode` manuel : `'json' => $data` encode et pose `Content-Type: application/json` tout seul.
- **`query` pour la query string**, jamais concat à la main dans l'URL : `'query' => ['page' => 1, 'filter' => 'x']` gère l'encodage (`urlencode`) et les valeurs nulles.
- **`auth_bearer` / `auth_basic`** plutôt qu'un header `Authorization` écrit à la main — plus lisible, évite les fautes de préfixe (`Bearer `, `Basic `).
- **`timeout` vs `max_duration`** : `timeout` = inactivité réseau (secondes sans byte reçu, défaut ∞). `max_duration` = durée totale de la requête (secondes mur-à-mur, défaut ∞). Pour une API tierce, mettre les deux (`timeout: 5`, `max_duration: 30`) sinon un serveur qui pond un octet toutes les 4 s bloque un worker indéfiniment.
- **`verify_peer: false` interdit en prod** : uniquement en dev local contre un certif auto-signé, et même là `cafile` est préférable. Ne jamais le pousser en `framework.yaml` racine.
- **Secrets** (`auth_bearer`, tokens, `auth_basic`) via `%env(SERVICE_TOKEN)%`, jamais en dur dans YAML ni dans le code.
- **Redirections** : 20 par défaut. Mettre `max_redirects: 0` sur les appels qui doivent échouer si l'URL n'est pas exacte (webhooks signés, callbacks OAuth).

## Choix du type de body

| Contenu                                    | Option                              | Content-Type posé             |
|--------------------------------------------|-------------------------------------|-------------------------------|
| JSON                                       | `'json' => $array`                  | `application/json`            |
| Form urlencoded                            | `'body' => $array`                  | `application/x-www-form-urlencoded` |
| Multipart (upload fichier + champs)        | `FormDataPart` → `body`+`headers`   | `multipart/form-data; boundary=…` |
| Payload brut (XML, binaire, texte)         | `'body' => $string`                 | celui des `headers` fourni    |
| Streaming sortant                          | `'body' => fopen(...)` ou closure   | celui des `headers` fourni    |

### Multipart avec fichier

```php
use Symfony\Component\Mime\Part\Multipart\FormDataPart;
use Symfony\Component\Mime\Part\DataPart;

$form = new FormDataPart([
    'metadata' => 'value',
    'file'     => DataPart::fromPath('/abs/path/to/file.pdf'),
]);

$response = $this->client->request('POST', '/upload', [
    'headers' => $form->getPreparedHeaders()->toArray(),
    'body'    => $form->bodyToIterable(),
]);
```

`bodyToIterable()` streame, `bodyToString()` charge tout en mémoire — préférer le premier pour les fichiers > quelques Mo. Nécessite `symfony/mime`.

## Déroulement

### 1 — Cadrer l'appel

- Quelle API ? URL de base, méthode d'auth, headers obligatoires (`Accept`, `User-Agent`).
- Appel récurrent (plusieurs endpoints de la même API) ou one-shot ? Récurrent → scoped client.
- Quels timeouts sont raisonnables côté API ? (si inconnu : `timeout: 5`, `max_duration: 30`).
- Body : JSON, form, multipart, brut ?

### 2 — Déclarer le scoped client (si récurrent)

`config/packages/framework.yaml` :

```yaml
framework:
    http_client:
        scoped_clients:
            github.client:
                base_uri: 'https://api.github.com/'
                headers:
                    Accept: 'application/vnd.github+json'
                    X-GitHub-Api-Version: '2022-11-28'
                auth_bearer: '%env(GITHUB_TOKEN)%'
                timeout: 5
                max_duration: 30
                max_redirects: 3
```

### 3 — Injecter et appeler

```php
use Symfony\Contracts\HttpClient\HttpClientInterface;

final class GithubClient
{
    public function __construct(
        private readonly HttpClientInterface $githubClient,
    ) {}

    public function fetchIssue(string $repo, int $number): array
    {
        $response = $this->githubClient->request('GET', "/repos/{$repo}/issues/{$number}", [
            'query' => ['expand' => 'comments'],
        ]);

        return $response->toArray();
    }
}
```

La gestion de réponse, des exceptions et du streaming → **`/symfony:http-client-response`**.

### 4 — Cas one-shot (pas de scoped client)

```php
$response = $this->client->request('POST', 'https://hooks.example.com/ingest', [
    'auth_bearer' => $this->parameterBag->get('ingest.token'),
    'json'        => $payload,
    'timeout'     => 3,
    'max_duration' => 10,
]);
```

Acceptable pour un appel isolé. Si un deuxième endpoint du même host apparaît, bascule en scoped client.

### 5 — Options moins courantes

- `'max_redirects' => 0` : désactive le suivi (utile pour OAuth callback, webhooks).
- `'proxy' => 'http://corp-proxy:3128'` + `'no_proxy' => 'localhost,.internal'` : proxy d'entreprise.
- `'on_progress' => fn(int $dl, int $size, array $info) => …` : callback de progression (téléchargement volumineux, annulation via exception).
- `'base_uri' => ['https://a/', 'https://b/']` : failover, combiné avec `RetryableHttpClient` (→ `/symfony:http-client-async`).
- `'user_data' => $id` : tag la réponse pour l'identifier lors du multiplexing (→ `/symfony:http-client-async`).
- `'extra' => ['curl' => [CURLOPT_… => …]]` : options cURL brutes, dernier recours.

### 6 — Vérification

```bash
symfony console debug:container --tag=http_client.client   # liste les clients enregistrés
symfony console debug:config framework http_client         # config effective
vendor/bin/phpstan analyse src
```

Tester l'appel réel une fois (dev ou sandbox) avec un logger actif — ensuite passer en tests mockés (→ `/symfony:http-client-test`).

### 7 — Clôture

Afficher :

- Scoped client ajouté / modifié dans `framework.yaml` (clé, `base_uri`, auth).
- Service créé / modifié, méthode publique exposée.
- Variables d'env attendues (`GITHUB_TOKEN`, …) à déclarer dans `.env` / `.env.test`.
- Ce qui reste : consommation de la réponse (→ `/symfony:http-client-response`), concurrence / retry (→ `/symfony:http-client-async`), tests (→ `/symfony:http-client-test`).

## Pièges fréquents

- **Pas d'erreur malgré un 500** : tant que `getStatusCode()` / `getHeaders()` / `getContent()` ne sont pas appelés, aucune exception ne part. La requête est asynchrone. Détail dans `/symfony:http-client-response`.
- **`body` + `json`** : mutuellement exclusifs. Si les deux sont passés, `json` l'emporte mais la config n'a pas de sens — choisir.
- **Header `Content-Type` posé à la main avec `'json' => …`** : écrase l'auto-détection et peut désaligner avec l'encodage réel. Laisser Symfony gérer.
- **`auth_basic` et `Authorization` header** coexistant : le header manuel gagne, la config `auth_basic` est silencieusement ignorée. Choisir un seul mécanisme.
- **Scoped client qui ne matche pas** : le `scope` (regex) est testé contre l'URL *absolue*. Si l'appel passe `/users` sans `base_uri`, aucun scope ne matche. Toujours utiliser le client injecté par nom plutôt que le client générique pour être sûr.

## Argument optionnel

`/symfony:http-client-request GithubClient` — cadre et scaffolde un service `GithubClient` + son scoped client.

`/symfony:http-client-request src/Client/StripeClient.php` — audite un client existant (scoped vs hardcoded, timeouts, secrets).

`/symfony:http-client-request` sans argument — demande l'API cible et le cas d'usage.

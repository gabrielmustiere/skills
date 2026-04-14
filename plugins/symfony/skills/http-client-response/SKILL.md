---
name: http-client-response
description: Consomme une réponse symfony/http-client — getContent, toArray, exceptions Client/Server/Transport. Déclenche sur "lire réponse HttpClient", "toArray", "getContent false". Impose try/catch des 3 exceptions.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /http-client-response — Consommer une réponse HttpClient

Tu consommes la réponse d'un appel `HttpClientInterface::request()` : lire le status, décoder le body, gérer les erreurs, ou streamer un gros téléchargement sans tout charger en mémoire.

## Détection préalable (obligatoire)

1. Lire `composer.json` et vérifier `symfony/http-client`. Absent → rediriger vers `/symfony:http-client-request` (installation).
2. Repérer l'endroit d'appel : `grep -rn '->request(' src/` ou repérer l'injection de `HttpClientInterface`. Sans appel identifié, demander quel service on instrumente.
3. Vérifier si la réponse est déjà consommée (`getStatusCode`, `getContent`, `toArray`, `stream`) ou si elle est renvoyée telle quelle — influence la stratégie d'erreur.

## Règles fondamentales

- **Rien ne bloque avant consommation** : `request()` retourne immédiatement une `ResponseInterface`, aucun byte n'est encore lu. La première méthode qui lit (headers ou body) bloque. Toute exception HTTP part à ce moment-là.
- **Pas de `getStatusCode()` seul pour "tester si ça marche"** : sans autre appel, la destruction de l'objet peut déclencher l'exception plus tard, dans un contexte confus. Soit on consomme le body (`getContent`/`toArray`), soit on annule explicitement (`$response->cancel()`).
- **`getContent(false)` ≠ "ignorer l'erreur"** : le `false` désactive l'exception sur 3xx/4xx/5xx, mais il faut **toujours** regarder `getStatusCode()` avant d'utiliser le contenu, sinon on traite un corps d'erreur comme une réponse valide.
- **`toArray()` décode le JSON en tableau PHP strict** (`assoc: true`). Lève `JsonException` si le JSON est invalide (via `DecodingExceptionInterface`). Utiliser `toArray(false)` pour ne pas lever sur 3xx/4xx/5xx et décoder quand même le body d'erreur.
- **Trois familles d'exceptions, trois interfaces** (et non des classes concrètes) :
  - `HttpExceptionInterface` : statut 3xx/4xx/5xx (`RedirectionException`, `ClientException`, `ServerException`).
  - `TransportExceptionInterface` : DNS, refus de connexion, TLS, timeout, câble arraché.
  - `DecodingExceptionInterface` : `toArray()` sur un corps non-JSON ou JSON cassé.

  Toujours catcher les interfaces, pas les classes — le composant peut évoluer sans casser le code.
- **`getHeaders()` déclenche la résolution des headers** (peut bloquer, peut lever). Les noms de header sont en lowercase dans le tableau retourné (`$headers['content-type'][0]`).
- **Streaming pour tout ce qui dépasse ~1 Mo** : `stream()` ou `toStream()`, jamais `getContent()` sur un fichier gros — ça charge tout en mémoire et peut OOM le worker.

## Déroulement

### 1 — Cas simple (JSON, < 1 Mo)

```php
use Symfony\Contracts\HttpClient\Exception\{
    ClientExceptionInterface,
    ServerExceptionInterface,
    TransportExceptionInterface,
    DecodingExceptionInterface,
};

try {
    $response = $this->githubClient->request('GET', "/repos/{$repo}");
    $data = $response->toArray();
} catch (ClientExceptionInterface $e) {            // 4xx : problème côté appelant
    $this->logger->warning('GitHub 4xx', ['status' => $e->getResponse()->getStatusCode()]);
    throw new RepoNotFoundException($repo, previous: $e);
} catch (ServerExceptionInterface $e) {            // 5xx : problème côté serveur → retry éventuel
    $this->logger->error('GitHub 5xx', ['body' => $e->getResponse()->getContent(false)]);
    throw new UpstreamUnavailableException(previous: $e);
} catch (TransportExceptionInterface $e) {         // réseau / TLS / timeout
    throw new UpstreamUnavailableException(previous: $e);
} catch (DecodingExceptionInterface $e) {          // body non-JSON
    throw new InvalidResponseException(previous: $e);
}
```

Ordre obligatoire : `ClientException` avant `ServerException` avant un catch générique. Les 3xx (`RedirectionException`) ne remontent que si `max_redirects: 0` — rares mais couvertes par `HttpExceptionInterface` si besoin.

### 2 — Cas "je veux gérer l'erreur sans exception"

```php
$response = $this->client->request('GET', $url);
$status = $response->getStatusCode();

if (200 !== $status) {
    $errorBody = $response->getContent(false);   // pas d'exception même si 4xx/5xx
    return $this->handleError($status, $errorBody);
}

return $response->toArray();                     // lève si 3xx/4xx/5xx (normal ici, déjà géré au-dessus)
```

Ce pattern est adapté quand l'erreur fait partie du flot métier (ex: 404 = "pas trouvé, return null"). Sinon, laisse les exceptions remonter : c'est le comportement par défaut pour une raison.

### 3 — Lire les headers

```php
$headers = $response->getHeaders();              // lève si 3xx/4xx/5xx
$contentType = $headers['content-type'][0] ?? null;

// Pagination GitHub
$link = $headers['link'][0] ?? null;
```

Clés toujours en lowercase, valeurs toujours `string[]` (même un header unique est un tableau).

Pour un seul header sans tolérance à l'erreur :

```php
$headers = $response->getHeaders(throw: false);
```

### 4 — Download volumineux via streaming

```php
use Symfony\Contracts\HttpClient\HttpClientInterface;

public function download(string $url, string $destPath): void
{
    $response = $this->client->request('GET', $url);

    if (200 !== $response->getStatusCode()) {
        throw new DownloadFailedException($url, $response->getStatusCode());
    }

    $fileHandle = fopen($destPath, 'w');
    try {
        foreach ($this->client->stream($response) as $chunk) {
            fwrite($fileHandle, $chunk->getContent());
        }
    } finally {
        fclose($fileHandle);
    }
}
```

Alternative plus concise via `toStream()` + `stream_copy_to_stream` :

```php
$source = $response->toStream();
$dest   = fopen($destPath, 'w');
stream_copy_to_stream($source, $dest);
fclose($dest);
```

Pour plusieurs téléchargements en parallèle → `/symfony:http-client-async`.

### 5 — Introspection via `getInfo()`

Utile en debug / metrics :

```php
$info = $response->getInfo();
// 'http_code', 'url', 'redirect_count', 'redirect_url',
// 'start_time', 'connect_time', 'total_time',
// 'response_headers' (brut), 'primary_ip', 'debug'
```

`getInfo('debug')` contient un log verbeux cURL-style — très utile pour comprendre pourquoi une requête échoue en dev.

### 6 — Annulation explicite

```php
$response = $this->client->request('GET', $url, ['on_progress' => $monitor]);
// ... décision métier ...
$response->cancel();   // libère le worker, pas d'exception
```

Si on veut juste jeter une réponse sans la consommer (et sans laisser partir une exception en destruction) : `$response->cancel()`.

### 7 — Clôture

Afficher :

- Service / méthode modifié.
- Stratégie d'erreur retenue (exceptions qui remontent vs `getContent(false)`).
- Exceptions métier levées par le service (`RepoNotFoundException`, `UpstreamUnavailableException`…) et leur mapping HTTP.
- Ce qui reste : retry / rate limit / multi-requêtes (→ `/symfony:http-client-async`), tests avec `MockHttpClient` (→ `/symfony:http-client-test`).

## Pièges fréquents

- **Exception qui sort en destruction** : `request()` puis le `ResponseInterface` sort de scope sans qu'on ait rien consommé → `TransportException` déclenchée plus tard avec une stack trace confuse. Fix : consommer ou `cancel()`.
- **`toArray()` sur une réponse non-JSON** : lève `JsonException`. Vérifier `content-type` ou catch `DecodingExceptionInterface`.
- **Boucle `foreach ($client->stream($response))` sans lire `$chunk->getContent()`** : la réponse ne progresse pas, timeout. Toujours consommer le chunk (ou `cancel()` pour arrêter net).
- **Headers en camelCase** : `$headers['Content-Type']` renvoie `null`. Utiliser `'content-type'`.
- **Mélange `throw: false` partiel** : `getHeaders(false)` + `toArray()` sans `false` → l'exception part sur le deuxième appel. Rester cohérent dans un même bloc.

## Argument optionnel

`/symfony:http-client-response src/Service/GithubClient.php` — audite la consommation de réponse dans ce service (try/catch, `toArray`, streaming, `cancel`).

`/symfony:http-client-response download` — scaffolde un téléchargement streamé (body → fichier).

`/symfony:http-client-response` sans argument — demande le service à instrumenter et le type de réponse attendu.

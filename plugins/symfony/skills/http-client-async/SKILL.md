---
name: http-client-async
description: Orchestre symfony/http-client — stream() multiplexing, Retryable/Caching/Throttling HttpClient, SSE. Déclenche sur "requêtes parallèles", "HttpClient concurrent", "retry HTTP". Impose fan-out + stream() et retry par decorator.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /http-client-async — Concurrence, retry, cache, rate limit, SSE

> **Utilise quand** tu lances plusieurs requêtes en parallèle, ajoutes retry/cache/rate-limit, ou consommes du SSE.
> **Pas quand** une seule requête séquentielle → `/symfony:http-client-request`.
> **Pas quand** tu cherches comment lire la réponse → `/symfony:http-client-response`.

Tu orchestres plusieurs requêtes HTTP sortantes en parallèle, tu ajoutes retry / cache / throttling sans polluer le code métier (via decorators), ou tu consommes un flux Server-Sent Events. Tu ne boucles jamais `request()` + `getContent()` séquentiellement quand N appels indépendants peuvent partir en parallèle.

## Détection préalable (obligatoire)

1. Lire `composer.json` et vérifier `symfony/http-client`. Absent → `/symfony:http-client-request`.
2. Pour le retry / le cache / le rate limit : les paquets sont optionnels.
   - Retry : inclus dans `symfony/http-client`. Rien à installer.
   - Cache HTTP : `composer require symfony/cache` (si pas déjà là).
   - Rate limiter : `composer require symfony/rate-limiter`.
   - SSE : inclus dans `symfony/http-client`.
3. Vérifier l'extension cURL. Sans cURL, le multiplexing fonctionne mais pas HTTP/2 : la concurrence est émulée via `NativeHttpClient`, les perfs sont moindres. Le signaler.

## Règles fondamentales

- **Pattern fan-out / fan-in** : envoyer toutes les `request()` d'abord (retour immédiat), consommer ensuite via `stream()`. Boucler `request(); getContent()` séquentiellement sérialise les appels — c'est le piège numéro un.
- **`stream()` traite dans l'ordre d'arrivée**, pas dans l'ordre d'envoi. Un endpoint rapide revient avant un lent, même s'il a été envoyé en deuxième. Pour recorréler : `'user_data' => $key` à l'envoi, `$response->getInfo('user_data')` à la consommation.
- **`max_host_connections`** (défaut 6) borne la concurrence **par host**. Pour N = 100 URLs sur le même host, les 6 premières partent, les suivantes attendent. Relever (`max_host_connections: 16`) uniquement si le serveur cible le permet — sinon on se fait throttler en retour.
- **Retry dans un decorator, pas dans le code métier** : `RetryableHttpClient` (ou la config `retry_failed` en bundle Symfony) gère l'exponential backoff, les idempotentes vs non-idempotentes, les codes 5xx et 429 automatiquement. Ne jamais écrire un `while (retry < 3)` à la main.
- **Cache HTTP via `CachingHttpClient`** (RFC 9111 : respecte `Cache-Control`, `ETag`, `Vary`) — pas un cache applicatif brut. Si l'API ne pose pas les bons headers, ce cache est neutre et il faut passer à un cache applicatif dans le service appelant.
- **Rate limit via `ThrottlingHttpClient`** (token bucket, fixed window, sliding window) — côté appelant pour ne pas déclencher le 429 du serveur. Complémentaire au retry (le retry gère les 429 qui passent quand même).
- **SSE via `EventSourceHttpClient`** : tient la connexion ouverte, reconnecte automatiquement. Consommer avec `stream()` et typer le chunk `instanceof ServerSentEvent` pour accéder à `getData()` / `getArrayData()`.
- **SSRF** : tout client qui suit une URL fournie par l'utilisateur (webhooks sortants, previews de liens) doit être enveloppé dans `NoPrivateNetworkHttpClient`, sinon l'attaquant peut atteindre `169.254.169.254` (cloud metadata), `localhost`, ou le LAN interne.

## Déroulement — Requêtes concurrentes

### 1 — Fan-out + fan-in basique

```php
public function fetchPackages(array $packages): array
{
    $responses = [];
    foreach ($packages as $name) {
        $responses[$name] = $this->client->request(
            'GET',
            "https://repo.packagist.org/p2/symfony/{$name}.json",
        );
    }

    $result = [];
    foreach ($responses as $name => $response) {
        $result[$name] = $response->toArray();   // consommation séquentielle mais requêtes parallèles
    }
    return $result;
}
```

### 2 — Multiplexing (ordre d'arrivée)

```php
$responses = [];
foreach ($packages as $idx => $name) {
    $responses[] = $this->client->request('GET', $url($name), ['user_data' => $name]);
}

foreach ($this->client->stream($responses) as $response => $chunk) {
    if ($chunk->isLast()) {
        $name = $response->getInfo('user_data');
        $this->bus->dispatch(new PackageFetched($name, $response->toArray()));
    }
}
```

Avantage : on traite chaque réponse dès qu'elle est complète, sans attendre la plus lente.

### 3 — Timeout sur le stream

```php
foreach ($this->client->stream($responses, 2.0) as $response => $chunk) {
    if ($chunk->isTimeout()) {
        $response->cancel();
        continue;
    }
    if ($chunk->isLast()) { /* ... */ }
}
```

Le 2.0 est un timeout d'inactivité *global* au batch — utile pour abandonner les lentes.

### 4 — Gestion d'erreur par réponse

```php
foreach ($this->client->stream($responses) as $response => $chunk) {
    try {
        if ($chunk->isFirst()) {
            $response->getStatusCode();   // déclenche HttpException si 4xx/5xx
        }
        if ($chunk->isLast()) { /* ... */ }
    } catch (TransportExceptionInterface | HttpExceptionInterface $e) {
        $this->logger->warning('Fetch failed', ['url' => $response->getInfo('url'), 'e' => $e]);
    }
}
```

Une erreur sur une réponse ne casse pas la boucle — les autres continuent.

## Déroulement — Decorators

### Retry (la bonne façon)

**Bundle Symfony** (préféré) — `framework.yaml` :

```yaml
framework:
    http_client:
        default_options:
            retry_failed:
                enabled: true
                max_retries: 3
                delay: 1000       # ms
                multiplier: 2     # exponentiel : 1s, 2s, 4s
                max_delay: 0      # 0 = pas de plafond
                jitter: 0.1
```

Par scoped client (granulaire) :

```yaml
framework:
    http_client:
        scoped_clients:
            flaky.api:
                base_uri: '…'
                retry_failed:
                    max_retries: 5
                    http_codes: [429, 502, 503, 504]
```

**Standalone / injection manuelle** :

```php
use Symfony\Component\HttpClient\RetryableHttpClient;
use Symfony\Component\HttpClient\Retry\GenericRetryStrategy;

$client = new RetryableHttpClient(
    HttpClient::create(),
    new GenericRetryStrategy(statusCodes: [423, 425, 429, 500, 502, 503, 504]),
    maxRetries: 3,
);
```

Point crucial : les retries sur méthodes non-idempotentes (POST) couvrent uniquement 423/425/429/502/503 par défaut — jamais 500/504 qui peuvent indiquer que la requête *est* passée. Ne pas élargir sans savoir si l'API est idempotente côté serveur.

### Cache RFC 9111

```yaml
framework:
    http_client:
        scoped_clients:
            docs.api:
                base_uri: 'https://docs.example.com/'
                caching:
                    cache_pool: docs_cache_pool

    cache:
        pools:
            docs_cache_pool:
                adapter: cache.adapter.redis_tag_aware
```

Se décide au niveau de chaque scoped client. Pas de cache global — chaque API a sa politique.

### Rate limiting

```yaml
framework:
    http_client:
        scoped_clients:
            stripe.api:
                base_uri: 'https://api.stripe.com/'
                auth_bearer: '%env(STRIPE_SECRET)%'
                rate_limiter: stripe_api

    rate_limiter:
        stripe_api:
            policy: 'token_bucket'
            limit: 100
            rate:
                interval: '1 second'
                amount: 100
```

Combiner avec `retry_failed` : le throttler limite le débit sortant, le retry gère les 429 qui passent quand même (burst côté serveur).

### SSRF

```php
use Symfony\Component\HttpClient\NoPrivateNetworkHttpClient;

$safe = new NoPrivateNetworkHttpClient($this->client);
$safe->request('GET', $userProvidedUrl);   // lève si URL pointe sur 10.0.0.0/8, 169.254/16, etc.
```

À appliquer sur tout appel où l'URL vient d'une entrée utilisateur. Liste de blocs custom en 2e argument si besoin.

## Déroulement — Server-Sent Events

```php
use Symfony\Component\HttpClient\EventSourceHttpClient;
use Symfony\Component\HttpClient\Chunk\ServerSentEvent;

$sse = new EventSourceHttpClient($this->client);
$source = $sse->connect('https://stream.example.com/events', [
    'headers' => ['Authorization' => 'Bearer '.$token],
]);

foreach ($sse->stream($source, 30) as $response => $chunk) {
    if ($chunk->isTimeout()) {
        continue;                  // pas d'event depuis 30s, on laisse respirer
    }
    if ($chunk instanceof ServerSentEvent) {
        $event = $chunk->getType();        // 'message' par défaut
        $data  = $chunk->getArrayData();   // si JSON
        $this->handle($event, $data);
    }
    if ($chunk->isLast()) {
        break;
    }
}
```

La reconnexion et l'en-tête `Last-Event-ID` sont gérés par le composant. Ne pas rouler sa propre reconnexion.

## Pièges fréquents

- **Consommation sérielle derrière un `request()` en boucle** : perte totale du gain de concurrence. Toujours envoyer avant de consommer.
- **`stream()` avec des réponses venant de deux clients différents** : `$client->stream($responses)` ne multiplexe que les réponses issues de *ce* client. Mixer `Psr18Client` et `HttpClient::create()` dans un même `stream()` casse.
- **`retry_failed` sur POST** : élargir `http_codes` à 500/504 ici peut causer un double-traitement côté serveur. Vérifier l'idempotence (clé d'idempotence Stripe, etc.).
- **`CachingHttpClient` "ne cache rien"** : l'API cible ne pose pas `Cache-Control` / `ETag`. Ce n'est pas un bug — c'est la spec. Passer à un cache applicatif.
- **SSE qui tue le worker** : un SSE doit tourner dans un process long (messenger worker, commande daemon), pas dans une requête HTTP front. Timeout PHP (`max_execution_time`) les tue sinon.
- **`NoPrivateNetworkHttpClient` autour de scoped_clients internes** : on bloque nos propres appels inter-services. N'appliquer la protection SSRF qu'aux clients qui consomment une URL utilisateur.

## Argument optionnel

`/symfony:http-client-async parallel` — scaffolde un fan-out + multiplexing.

`/symfony:http-client-async retry` — configure `retry_failed` sur un scoped client existant.

`/symfony:http-client-async rate-limit` — branche `ThrottlingHttpClient` sur un client.

`/symfony:http-client-async sse` — scaffolde un consommateur SSE.

`/symfony:http-client-async` sans argument — demande le cas d'usage (parallèle, retry, cache, rate limit, SSE, SSRF).

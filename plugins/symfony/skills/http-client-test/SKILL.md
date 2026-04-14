---
name: http-client-test
description: Teste un service HttpClientInterface — MockHttpClient, MockResponse, JsonMockResponse, HAR. Déclenche sur "tester HttpClient", "MockHttpClient", "JsonMockResponse". Impose le mock, jamais d'appel réseau réel.
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /http-client-test — Tester un service qui consomme HttpClientInterface

Tu testes un service qui consomme `HttpClientInterface` sans jamais toucher au réseau. Tu injectes `MockHttpClient`, tu fixes les réponses attendues, tu assertes à la fois le résultat du service et le contenu des requêtes sortantes.

## Détection préalable (obligatoire)

1. Lire `composer.json`. Vérifier `symfony/http-client`. Absent → `/symfony:http-client-request`.
2. Repérer le framework de test : `phpunit/phpunit` (unit / integration), `symfony/framework-bundle` (`KernelTestCase`, `WebTestCase`). Adapter les exemples à celui présent.
3. Repérer le service cible : `grep -rn 'HttpClientInterface' src/` — confirmer qu'il est bien injecté par constructeur (sinon le mock n'a pas de prise).

## Règles fondamentales

- **Aucun appel réseau en test** : unit *et* intégration. Chaque test qui sort sur Internet est un test flaky en puissance (DNS, TLS, rate limit du CI, secrets). Sans exception.
- **`MockHttpClient` injecté à la place de `HttpClientInterface`** : en unit, construction directe du service avec le mock. En intégration (`KernelTestCase`), remplacer le service via `self::getContainer()->set('...')` ou via la config `when@test`.
- **`MockResponse` pour la plupart des cas**, `JsonMockResponse` quand on renvoie du JSON (plus ergonomique : accepte un array, pose `Content-Type`), callback (`fn ($method, $url, $options) => …`) quand la réponse dépend de la requête (router de mocks).
- **Tester les deux côtés** : le retour du service (comportement) *et* les requêtes émises (URL, méthode, headers, body). `MockResponse::getRequestMethod/Url/Options()` expose ce qui a été envoyé.
- **Ordre des réponses = ordre des appels** : avec une liste de `MockResponse`, le i-ème mock répond au i-ème appel. Si le service fait plus d'appels que prévu, les tests lèvent une exception *« the mock response factory is exhausted »* — utile pour détecter les appels non voulus.
- **Pas de sleep, pas de retry réel en test** : si le service utilise `RetryableHttpClient`, le test doit soit désactiver le retry (`retry_failed.enabled: false` en `when@test`), soit fournir une séquence de mocks qui simule les échecs puis le succès — sinon le test attend les backoffs réels.
- **Fixtures JSON externalisées** : dès qu'un body dépasse 10 lignes, le sortir dans `tests/fixtures/<service>/<cas>.json` et charger via `JsonMockResponse::fromFile(...)`. Lisibilité et reuse.

## Déroulement — Test unitaire

### 1 — Service minimal à tester

```php
final class GithubClient
{
    public function __construct(private readonly HttpClientInterface $githubClient) {}

    public function repoName(string $ownerRepo): string
    {
        return $this->githubClient->request('GET', "/repos/{$ownerRepo}")->toArray()['name'];
    }
}
```

### 2 — Test "happy path"

```php
use PHPUnit\Framework\TestCase;
use Symfony\Component\HttpClient\MockHttpClient;
use Symfony\Component\HttpClient\Response\JsonMockResponse;

final class GithubClientTest extends TestCase
{
    public function testRepoName(): void
    {
        $mock = new JsonMockResponse(['name' => 'symfony']);
        $client = new MockHttpClient($mock, 'https://api.github.com');

        $sut = new GithubClient($client);
        self::assertSame('symfony', $sut->repoName('symfony/symfony'));

        self::assertSame('GET', $mock->getRequestMethod());
        self::assertSame('https://api.github.com/repos/symfony/symfony', $mock->getRequestUrl());
    }
}
```

Le 2e argument de `MockHttpClient` est la `base_uri` — essentiel pour que `getRequestUrl()` renvoie l'URL absolue attendue.

### 3 — Assertions sur le body et les headers

```php
$mock = new JsonMockResponse(['id' => 42], ['http_code' => 201]);
$client = new MockHttpClient($mock);

$sut = new IssueCreator($client);
$sut->create('symfony/symfony', 'Bug', 'Describe');

$options = $mock->getRequestOptions();
self::assertContains('Content-Type: application/json', $options['headers']);
self::assertContains('Authorization: Bearer test-token', $options['headers']);

$body = json_decode($options['body'], true, flags: JSON_THROW_ON_ERROR);
self::assertSame(['title' => 'Bug', 'body' => 'Describe'], $body);
```

`getRequestOptions()` retourne les options **normalisées** : les headers y sont sous forme `'Name: value'`, et le body sous forme string (l'encodage JSON a déjà été fait).

### 4 — Test d'erreur HTTP

```php
$mock = new JsonMockResponse(['message' => 'Not Found'], ['http_code' => 404]);
$client = new MockHttpClient($mock);

$this->expectException(RepoNotFoundException::class);
(new GithubClient($client))->repoName('ghost/repo');
```

Le service doit catcher `ClientExceptionInterface` et mapper vers son exception métier — sinon l'exception transport HttpClient remonte.

### 5 — Séquence de réponses

```php
$mocks = [
    new JsonMockResponse(['token' => 'abc']),                          // login
    new JsonMockResponse(['items' => [1, 2, 3]]),                      // fetch
    new JsonMockResponse([], ['http_code' => 204]),                    // delete
];
$client = new MockHttpClient($mocks);
```

Le `MockHttpClient` consomme la queue dans l'ordre. Si le code fait 4 appels, exception à la 4e. Utile pour détecter les régressions qui introduisent un appel en trop.

### 6 — Callback dynamique

```php
$client = new MockHttpClient(
    function (string $method, string $url, array $options): MockResponse {
        return match (true) {
            str_contains($url, '/login')   => new JsonMockResponse(['token' => 'abc']),
            str_contains($url, '/items')   => new JsonMockResponse(['items' => [1, 2]]),
            default                         => new MockResponse('', ['http_code' => 404]),
        };
    },
);
```

Préférer cette forme quand l'ordre d'appel n'est pas déterministe ou que plusieurs tests partagent le même routeur.

### 7 — Streaming / body en générateur

```php
$body = function (): \Generator {
    yield "chunk-1\n";
    yield '';           // simule un timeout intermédiaire (isTimeout() true)
    yield "chunk-2\n";
};
$mock = new MockResponse($body(), ['response_headers' => ['Content-Type' => 'text/plain']]);
```

Indispensable pour tester un service qui consomme `stream()`.

## Déroulement — Test d'intégration (KernelTestCase)

### 1 — Via `when@test` et un factory

`config/services_test.yaml` :

```yaml
services:
    Symfony\Contracts\HttpClient\HttpClientInterface:
        class: Symfony\Component\HttpClient\MockHttpClient
        arguments:
            $responseFactory: '@App\Tests\Double\GithubMockFactory'
            $baseUri: 'https://api.github.com'
        public: true
```

```php
namespace App\Tests\Double;

use Symfony\Component\HttpClient\Response\{JsonMockResponse, MockResponse};

final class GithubMockFactory
{
    public function __invoke(string $method, string $url, array $options): MockResponse
    {
        return new JsonMockResponse(['name' => 'symfony']);
    }
}
```

Tous les tests d'intégration partagent ce factory — il doit rester simple. Pour les cas spécifiques, override par test via `self::getContainer()->set(...)`.

### 2 — Override par test

```php
final class FetchRepoTest extends KernelTestCase
{
    public function testFetch(): void
    {
        self::bootKernel();
        $mock = new JsonMockResponse(['name' => 'custom']);
        self::getContainer()->set(HttpClientInterface::class, new MockHttpClient($mock));

        $sut = self::getContainer()->get(GithubClient::class);
        self::assertSame('custom', $sut->repoName('x/y'));
    }
}
```

En Symfony 6.2+, les services injectés par autowiring sont bien substitués par ce mécanisme à condition que le service soit marqué `public: true` (ou récupéré via `static::getContainer()->set(...)` qui bypasse la restriction).

### 3 — Scoped client nommé

Si le service injecte `$githubClient` (scoped), l'alias dans `framework.yaml` génère `http_client.scoped.github.client` — le remplacer :

```php
self::getContainer()->set('github.client', new MockHttpClient($mock));
```

ou déclarer l'override dans `services_test.yaml` avec la même clé.

## Déroulement — Fixtures HAR

Pour rejouer un flux complet enregistré depuis DevTools :

1. DevTools → onglet **Network** → clic droit → **Save all as HAR with content**.
2. Enregistrer dans `tests/fixtures/<scenario>.har`.
3. Consommer :

```php
use Symfony\Component\HttpClient\Response\MockResponse;

$responses = MockResponse::fromHar(__DIR__.'/fixtures/login-flow.har');
$client    = new MockHttpClient($responses);
```

Les fichiers HAR contiennent cookies et auth — les assainir avant commit.

## Déroulement — Profiling (TraceableHttpClient)

Utile en debug de test, pas en prod :

```php
use Symfony\Component\HttpClient\TraceableHttpClient;

$traceable = new TraceableHttpClient($client);
$sut = new GithubClient($traceable);
$sut->repoName('x/y');

foreach ($traceable->getTracedRequests() as $trace) {
    dump($trace['method'].' '.$trace['url'], $trace['options']);
}
```

Ne laisse pas `dump()` dans un test — utilitaire ponctuel pour comprendre pourquoi une assertion échoue.

## Pièges fréquents

- **`MockHttpClient` sans `base_uri`** : `getRequestUrl()` retourne l'URL relative passée au `request()`, ce qui rend les assertions fragiles. Toujours passer `base_uri` (2e argument ou via `request` absolu).
- **Réponse JSON avec `MockResponse` au lieu de `JsonMockResponse`** : oubli du header `Content-Type`, `toArray()` marche quand même mais les services qui testent ce header cassent. Utiliser `JsonMockResponse`.
- **Mock trop permissif** : callback qui renvoie toujours un 200 → masque les bugs d'URL. Assertions strictes sur la méthode et l'URL, ou séquence ordonnée.
- **Test qui dépend de `retry_failed`** : les backoffs réels ralentissent le test et le rendent flaky. Désactiver en `when@test` ou simuler la séquence.
- **HAR jamais assaini** : tokens, cookies de session committés dans le repo. Script de purge avant commit.

## Argument optionnel

`/symfony:http-client-test tests/Client/GithubClientTest.php` — audit / complète un test existant (assertions manquantes, absence de `base_uri`, mocks trop laxistes).

`/symfony:http-client-test GithubClient` — scaffolde un `GithubClientTest` unit + integration pour le service indiqué.

`/symfony:http-client-test` sans argument — demande le service à tester et le type (unit, integration, HAR).

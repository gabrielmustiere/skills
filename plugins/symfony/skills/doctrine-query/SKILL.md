---
name: doctrine-query
description: Écrit ou optimise une requête Doctrine (find/findBy, DQL, QueryBuilder, DBAL) — Symfony/Sylius. Déclenche sur "requête Doctrine", "repository custom", "QueryBuilder", "DQL", "JOIN fetch", "N+1", "optimiser requête".
user_invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# /doctrine-query — Requêtes Doctrine dans les repositories

Tu conçois ou révises une requête Doctrine. Tu choisis **le bon outil** pour le contexte, tu isoles la requête dans un repository, et tu anticipes les N+1.

## Détection préalable

Lire `composer.json` ; vérifier `symfony/framework-bundle`. Absent → demander confirmation avant de continuer. Mentionner Sylius en une ligne si `sylius/sylius` présent (repositories doivent filtrer par channel sur données channel-scopées).

## Choix de l'outil

Quel que soit le cas, le résultat est **toujours exposé via une méthode nommée du repository** (ex: `findActiveByOwner`, `countPendingForChannel`). L'outil ci-dessous est le moyen d'implémentation **à l'intérieur** de cette méthode — pas un raccourci à appeler directement depuis un service.

| Cas                                                             | Outil                      | Pourquoi                                             |
|-----------------------------------------------------------------|----------------------------|------------------------------------------------------|
| Lookup par ID ou critères simples (`status = 'open'`)           | `find`, `findBy`, `findOneBy` (appelés depuis la méthode repo custom) | Zéro code, déjà couvert par `EntityRepository` — mais wrappé dans une méthode nommée |
| Critères fixes avec tri / limite                                | DQL                        | Lisible, paramétré, typé entité                      |
| Critères **dynamiques** (formulaire de recherche, filtres admin) | QueryBuilder               | Construction conditionnelle propre, pas de `sprintf` |
| Agrégats complexes, fonctions SGBD, perf critique               | SQL brut via DBAL          | Contrôle total, mais on perd l'hydratation entité    |
| Lecture d'une entité depuis un paramètre de route               | `EntityValueResolver`      | `#[MapEntity]` ou injection directe dans le contrôleur — seul cas légitime où un service appelant n'invoque pas explicitement une méthode repo (le resolver le fait pour lui) |

## Règles fondamentales

- **Tout accès aux données passe par une méthode nommée d'un repository.** Jamais `$em->find(...)`, `$em->getRepository(...)->findBy(...)`, ni QueryBuilder construit ailleurs qu'un repository — que ce soit dans un contrôleur, un service, un listener, un Twig extension, un command handler. L'extérieur appelle `OrderRepository::findPendingForUser($user)`, pas `findBy(['status' => 'pending', 'user' => $user])`. La méthode expose l'**intention métier**, pas la mécanique SQL ; elle est testable unitairement, typée, réutilisable, et l'optimisation (JOIN fetch, index) se fait à un seul endroit.
- **Tout DQL/SQL vit dans `src/Repository/`**. Corollaire direct de la règle ci-dessus — aucun DQL ni SQL brut n'a le droit d'apparaître hors repository.
- **Paramètres nommés** (`:userId`), jamais de concaténation. Paramètre binding = protection SQLi + cache de plan.
- **`fetch JOIN` pour les relations qu'on va itérer**. Un `foreach` sur une relation lazy = N+1. Trace : regarder le profiler / activer `doctrine.dbal.logging` en dev.
- **Pagination obligatoire** sur toute liste potentiellement grande (admin grids, API, exports). Utiliser `Paginator` pour conserver le comptage correct avec JOIN.
- **Pas d'appel repository dans un `foreach`**. Charger en batch en amont via une méthode repo dédiée (`findByIds(array $ids)`).

## Patterns canoniques

### Repository custom avec QueryBuilder

```php
public function findActiveByOwner(User $owner, int $limit = 20): array
{
    return $this->createQueryBuilder('p')
        ->andWhere('p.owner = :owner')
        ->andWhere('p.status = :status')
        ->setParameter('owner', $owner)
        ->setParameter('status', Status::Active)
        ->orderBy('p.createdAt', 'DESC')
        ->setMaxResults($limit)
        ->getQuery()
        ->getResult();
}
```

### DQL avec JOIN fetch (anti-N+1)

```php
public function findWithItems(int $orderId): ?Order
{
    return $this->getEntityManager()->createQuery('
        SELECT o, i
        FROM App\Entity\Order o
        JOIN FETCH o.items i
        WHERE o.id = :id
    ')->setParameter('id', $orderId)
      ->getOneOrNullResult();
}
```

### Pagination sans N+1

```php
use Doctrine\ORM\Tools\Pagination\Paginator;

$qb = $this->createQueryBuilder('p')->leftJoin('p.tags', 't')->addSelect('t');
$paginator = new Paginator($qb->getQuery()->setFirstResult($offset)->setMaxResults($limit));
return ['items' => iterator_to_array($paginator), 'total' => count($paginator)];
```

### SQL brut via DBAL (cas d'échappement)

```php
$conn = $this->getEntityManager()->getConnection();
$rows = $conn->executeQuery(
    'SELECT id, total_cents FROM orders WHERE channel_id = :channel',
    ['channel' => $channelId],
)->fetchAllAssociative();
```

### EntityValueResolver dans un contrôleur

```php
#[Route('/product/{slug}')]
public function show(#[MapEntity(mapping: ['slug' => 'slug'])] Product $product): Response
{ /* ... */ }
```

`?Product $product` en nullable désactive le 404 auto.

## Delta Sylius

Si `sylius/sylius` présent :

- Données channel-scopées → **toujours** filtrer `->andWhere('x.channel = :channel')` dans le repository, sinon fuite entre channels.
- Repositories Sylius étendus : hériter du repository de base + déclaration dans `_sylius.yaml` sous `sylius_<bundle>.resources.<x>.classes.repository`.

## Déroulement

### 1 — Contexte

- Quelle entité ? quel cas d'usage (admin grid, page shop, API, export) ?
- Critères fixes ou dynamiques ?
- Relation(s) à charger ?
- Volume attendu ?

### 2 — Écriture

Ouvrir le repository concerné (`src/Repository/<Entity>Repository.php`), ajouter la méthode avec le bon outil selon la table ci-dessus.

### 3 — Vérification perf

```bash
symfony console doctrine:query:dql 'SELECT ...'          # tester un DQL
vendor/bin/phpstan analyse src/Repository                 # types
```

En dev, activer le panel Doctrine du profiler et compter les queries — un JOIN fetch qui baisse de N+1 à 1 est le signe que c'est gagné.

### 4 — Tests

- **Functional test** (BDD de test) sur les repositories custom : un test par méthode non triviale.
- Fixtures reproductibles — pas de dépendance à l'ordre.

### 5 — Clôture

Afficher :

- Méthode(s) ajoutée(s), outil choisi.
- Nombre de queries avant/après si optimisation.
- Tests associés (ou suggestion de les écrire).

## Argument optionnel

`/symfony:doctrine-query OrderRepository findPending` — ajoute/révise la méthode `findPending` dans `OrderRepository`.

`/symfony:doctrine-query src/Repository/OrderRepository.php` — audit de tout le fichier (N+1, pagination, paramètres nommés).

`/symfony:doctrine-query` sans argument — demande le repository et le cas d'usage.

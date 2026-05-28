# Inventaire — plugin `symfony`

Skills Symfony/Doctrine groupées par domaine (25 skills).

| Skill | Rôle |
| --- | --- |
| [`doctrine-entity`](../plugins/symfony/skills/doctrine-entity/SKILL.md) | Entité Doctrine : ORM, relations, types custom |
| [`doctrine-query`](../plugins/symfony/skills/doctrine-query/SKILL.md) | Requête Doctrine : find/findBy, DQL, QueryBuilder, DBAL |
| [`doctrine-migration`](../plugins/symfony/skills/doctrine-migration/SKILL.md) | Migration : génération, dry-run, réversibilité |
| [`event-dispatch`](../plugins/symfony/skills/event-dispatch/SKILL.md) | Définit et dispatche un événement custom |
| [`event-listen`](../plugins/symfony/skills/event-listen/SKILL.md) | Event Listener (`#[AsEventListener]`) |
| [`event-subscribe`](../plugins/symfony/skills/event-subscribe/SKILL.md) | Event Subscriber (multi-callbacks, priorités) |
| [`form-type`](../plugins/symfony/skills/form-type/SKILL.md) | Classe FormType (AbstractType, buildForm) |
| [`form-handle`](../plugins/symfony/skills/form-handle/SKILL.md) | Traitement en contrôleur (createForm, handleRequest, PRG) |
| [`form-render`](../plugins/symfony/skills/form-render/SKILL.md) | Rendu Twig (thèmes, customisation par bloc) |
| [`form-advanced`](../plugins/symfony/skills/form-advanced/SKILL.md) | DataTransformer, FormEvents, CollectionType, FileType |
| [`controller-action`](../plugins/symfony/skills/controller-action/SKILL.md) | Contrôleur fin (AbstractController, helpers, `#[MapQueryParameter]`, `#[MapRequestPayload]`) |
| [`routing-define`](../plugins/symfony/skills/routing-define/SKILL.md) | `#[Route]` (path, methods, requirements, host, locale), génération d'URL, URIs signées |
| [`http-client-request`](../plugins/symfony/skills/http-client-request/SKILL.md) | Construction d'une requête HTTP sortante |
| [`http-client-response`](../plugins/symfony/skills/http-client-response/SKILL.md) | Consommation de réponse, exceptions, streaming |
| [`http-client-async`](../plugins/symfony/skills/http-client-async/SKILL.md) | Concurrence, retry, cache, rate-limit, SSE |
| [`http-client-test`](../plugins/symfony/skills/http-client-test/SKILL.md) | Mock HttpClient, MockResponse, HAR |
| [`messenger-async`](../plugins/symfony/skills/messenger-async/SKILL.md) | Message + handler `#[AsMessageHandler]`, transports (doctrine/amqp/redis), retry, DLQ, worker |
| [`serializer-use`](../plugins/symfony/skills/serializer-use/SKILL.md) | `SerializerInterface`, normalizers/encoders (json/xml/csv), `#[Groups]`, `#[Context]`, `#[DiscriminatorMap]` |
| [`object-mapper`](../plugins/symfony/skills/object-mapper/SKILL.md) | Symfony ObjectMapper 7.3+ : `#[Map]` (target/source/if/transform), conditions, transformers, MapCollection |
| [`service-define`](../plugins/symfony/skills/service-define/SKILL.md) | Déclaration (autowiring, autoconfiguration) |
| [`service-wire`](../plugins/symfony/skills/service-wire/SKILL.md) | Câblage d'arguments (scalaires, env, alias, `#[Target]`) |
| [`service-tags`](../plugins/symfony/skills/service-tags/SKILL.md) | Tags, collections, factories, décorateurs |
| [`validation-constraints`](../plugins/symfony/skills/validation-constraints/SKILL.md) | Contraintes `#[Assert\*]` sur entité/DTO/classe |
| [`validation-groups`](../plugins/symfony/skills/validation-groups/SKILL.md) | Groupes, séquences, validation conditionnelle |
| [`validation-use`](../plugins/symfony/skills/validation-use/SKILL.md) | `ValidatorInterface` hors form (DTO, payload, CLI) |

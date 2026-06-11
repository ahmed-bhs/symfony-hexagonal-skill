# Dependency Rules

## Import Rules by Layer

### Domain Layer — ZERO external dependencies

```
ALLOWED imports in Domain/:
✅ Other Domain/ classes (same or different module)
✅ PHP built-in classes (DateTimeImmutable, InvalidArgumentException, etc.)
✅ Declarative mapping/validation attributes (inert metadata):
     #[Doctrine\ORM\Mapping\...], format #[Symfony\...\Validator\Constraints\...]
     (NotBlank, Email, Length, Url, NotCompromisedPassword…)

FORBIDDEN imports in Domain/:
❌ Doctrine\* services (EntityManager, Connection, QueryBuilder…)
❌ Symfony\* services (HttpClient, Mailer, Security services…)
❌ DB-access constraints: #[UniqueEntity], #[Assert\Callback], #[Assert\Expression]
❌ App\*\Application\*  /  App\*\Infrastructure\*
❌ Any third-party library logic
```

> Domain forbids framework *behaviour*, not framework *metadata*. `#[ORM\...]`
> and format `#[Assert\...]` are inert attributes — they keep the entity
> testable as plain PHP (`new User(...)` works without a kernel) while avoiding
> the cost of external XML mapping. Doctrine imposes the same runtime
> constraints (non-final, reflection hydration) whether you map via attributes
> or XML, so externalising the mapping buys hygiene, not capability. The hard
> line is DB-access constraints: `UniqueEntity`/`Callback`/`Expression` run a
> query or arbitrary service code and must live in `Adapter/In/Validation/`.

### Application Layer

```
ALLOWED imports in Application/:
✅ App\{Module}\Domain\* (entities, value objects, ports, events)
✅ App\Shared\Application\Port\In\* (the buses)
✅ Framework vendors used as orchestration tools (Messenger, Validator)

FORBIDDEN imports in Application/:
❌ App\*\Infrastructure\*
❌ Doctrine\*
❌ Concrete adapter classes
```

> Prefer a framework-free Application: handlers implement a neutral marker
> interface and receive the bus as a port, Messenger wiring done in
> `services.yaml`. Using `#[AsMessageHandler]` is an accepted pragmatic
> alternative — pick one per project.

### Infrastructure Layer — Adapters

```
ALLOWED imports in Infrastructure/Adapter/:
✅ App\{Module}\Domain\*       (implements driven port interfaces)
✅ App\{Module}\Application\*  (driving adapters dispatch use cases via the bus)
✅ Doctrine\*, Symfony\*, third-party SDKs

Direction:
  Adapter/In  -> Application -> Domain
  Adapter/Out -> implements Domain/Application ports
```

There is no `Presentation` layer. Driving adapters (`Adapter/In/`) are the only
place allowed to depend on Application *and* the framework's HTTP/CLI/security
stack.

## Deptrac Configuration

Real-world enforcement uses directory collectors keyed on the layer name inside
each module path (module-first). Layers are matched by the `/Domain/`,
`/Application/`, `/Infrastructure/` segment, regardless of which module.

```yaml
# deptrac.yaml
deptrac:
    paths:
        - ./src/User
        - ./src/Shared
        # ... one entry per bounded context
    layers:
        - name: Domain
          collectors:
              - type: directory
                value: .+/Domain/.*
        - name: Application
          collectors:
              - type: directory
                value: .+/Application/.*
        - name: Infrastructure
          collectors:
              - type: directory
                value: .+/Infrastructure/.*

        # Mapping/validation libs whose attributes are allowed in Domain.
        # Inert metadata only — the entity stays plain-PHP testable.
        - name: MappingVendors
          collectors:
              - type: composer
                composerPath: composer.json
                composerLockPath: composer.lock
                packages:
                    - doctrine/orm
                    - symfony/validator
                    - symfony/serializer
        - name: Vendors
          collectors:
              - type: composer
                composerPath: composer.json
                composerLockPath: composer.lock
                packages:
                    - symfony/messenger
                    # ... other runtime libs allowed in App/Infra

    ruleset:
        Domain:
            - MappingVendors            # inert ORM/validation attributes only
        Application:
            - Domain
            - MappingVendors
            - Vendors
        Infrastructure:
            - Domain
            - Application
            - MappingVendors
            - Vendors

        # DB-access constraints stay OUT of Domain — declare a forbidden layer
        # and exclude it from the Domain ruleset.
        - name: ForbiddenDomainConstraints
          collectors:
              - type: classNameRegex
                value: '/UniqueEntity$|Constraints\\Callback$|Constraints\\Expression$/'
```

> Domain depends on `MappingVendors` (Doctrine ORM + Validator attributes) — this
> is the inert metadata exception. It does NOT depend on `Vendors` (runtime
> services) nor on `ForbiddenDomainConstraints` (DB-access validators), which
> belong in `Adapter/In/Validation/`.

## PHPStan Configuration (phpstan.neon)

```neon
parameters:
    level: 8
    paths:
        - src/
    excludePaths:
        - src/Kernel.php

includes:
    - vendor/phpstan/phpstan-symfony/extension.neon
    - vendor/phpstan/phpstan-doctrine/extension.neon
```

```
composer require --dev \
    phpstan/phpstan \
    phpstan/phpstan-symfony \
    phpstan/phpstan-doctrine \
    phpstan/phpstan-strict-rules \
    phpstan/phpstan-deprecation-rules
```

## Cross-Module Communication

Modules communicate through domain events, not direct imports:

```
ALLOWED cross-module:
✅ Module A dispatches event → Module B handles event
✅ Shared kernel value objects (App\Shared\Domain\*)

FORBIDDEN cross-module:
❌ Module A directly calls Module B's repository
❌ Module A imports Module B's entities
❌ Direct service-to-service calls between modules
```

### Shared Module

Common value objects, contracts and the buses shared across modules:

```
src/Shared/
├── Domain/
│   ├── ValueObject/
│   │   ├── AggregateRootId.php          # base id for aggregates
│   │   ├── Uuid.php
│   │   ├── Money.php
│   │   └── DateRange.php
│   ├── Event/DomainEventInterface.php   # base domain-event interface
│   ├── Trait/{Timestampable,EventRecorder}Trait.php
│   └── Contract/{CrossCuttingInterface}.php
├── Application/
│   ├── Port/In/{Command,Query,Event}BusInterface.php   # driving ports
│   └── Bus/{Command,Query}HandlerInterface.php         # markers
└── Infrastructure/Adapter/Out/Bus/Messenger*Bus.php    # bus adapters
```

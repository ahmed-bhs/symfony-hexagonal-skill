---
description: "Symfony ports and adapters — port interfaces, adapter implementations, dependency injection, autowiring, repository interfaces. Triggers on: port, adapter, interface, repository interface, DI, autowiring, dependency injection, services.yaml, binding"
---

# Symfony Ports & Adapters

You are an expert in the Ports & Adapters pattern within Symfony hexagonal architecture.

## When to Activate

- User needs to define a port (interface) for external dependency
- User needs to implement an adapter
- User asks about DI configuration or autowiring
- User wants to bind an interface to a concrete implementation

## Port = Interface in Domain

Ports are interfaces that define contracts for external dependencies. Driven (repository/gateway) contracts live in `Domain/Port/Out/`; technical orchestration ports live in `Application/Port/Out/`.

### Rules
- One interface per concern (Single Responsibility)
- Domain-centric naming (not tech-centric): `UserRepositoryInterface` not `DoctrineUserRepository`
- Only domain types in signatures (value objects, entities, primitives)
- No framework types in port signatures

### Common Port Types

| Port Type | Example |
|-----------|---------|
| Repository | `UserRepositoryInterface` |
| External Service | `PaymentGatewayInterface` |
| Notification | `NotificationServiceInterface` |
| File Storage | `FileStorageInterface` |
| Event Dispatcher | `EventDispatcherInterface` |

### Template
```php
namespace App\{Module}\Domain\Port\Out;

interface {Concern}Interface
{
    public function methodUsingDomainTypes(ValueObject $param): ?Entity;
}
```

## Adapter = Implementation in Infrastructure

Adapters implement ports using specific technologies. Driven adapters live in `Infrastructure/Adapter/Out/{Doctrine,Storage,Bus,...}`; driving adapters live in `Infrastructure/Adapter/In/{ApiPlatform,CLI,Admin,Security,Validation}`.

### Rules
- Implements exactly one port interface
- Contains ALL technology-specific code
- Named with technology prefix: `Doctrine{Entity}Repository`, `Stripe{Payment}Gateway`
- `final readonly class` when possible

### Template
```php
namespace App\{Module}\Infrastructure\Adapter\Out\Doctrine;

use App\{Module}\Domain\Port\Out\{Repository}Interface;

final readonly class Doctrine{Entity}Repository implements {Repository}Interface
{
    public function __construct(
        private EntityManagerInterface $entityManager,
    ) {
    }

    // implement all port methods
}
```

## Port Placement: Domain vs Application

The location of a port Out depends on **who expresses the need**, not on the types in its signature. Typing is a hint but not proof — a signature can manipulate domain types while the need is actually an application concern.

| The need originates in... | Port goes in |
|---------------------------|--------------|
| the domain (a domain concept requires it) | `Domain/Port/Out/` |
| a use case (orchestration/technical need) | `Application/Port/Out/` |

## Driving vs Driven Adapters and Ports

A canonical asymmetry of Ports & Adapters — get this right:

- **Adapter Out (driven)** — **implements** a port Out. The application needs an external service (DB, storage, computation); the contract is pulled by the application through a Domain interface, and the adapter provides the technical implementation. The dependency arrow points from the application to the port, never to the technology.
  - Example: `DoctrineOrderRepository implements OrderRepositoryInterface`.

- **Adapter In (driving)** — does **NOT** implement a port; it **consumes** a port In by calling it. It drives the application, so it cannot implement what it drives. The port In is the bus (`QueryBusInterface` / `CommandBusInterface`); the In adapter depends on it and pushes a typed message. The bus is implemented in Infrastructure (`MessengerQueryBus`), which routes to the handler chosen by the message type.
  - Example: a controller/live component depends on `QueryBusInterface` and sends `ListOrdersQuery` — it implements no business port.

Consequence: the link `Adapter In → use case` is carried by the **message type**, not by a dedicated interface. Layer traceability is carried by the structure itself — enforce it with deptrac when configured (forbid `Domain → Application`); the bundled `symfony-leaky-abstractions` skill and reviewer catch the common cases via static scan.

## Naming: Repository vs Search/Finder, and Suffixes

`Repository` and `Search`/`Finder` are both port Out (`Port/Out/`). The name conveys the role, not the port status.

- Name **`Repository`** when the returned object is a rich aggregate (loaded by identity to be mutated).
- Name **`Search`/`Finder`** when the returned object is a read projection/DTO.

Suffix convention:
- Ports carry the `Interface` suffix (not `Port`) — port status is already conveyed by the `Port/Out/` location.
- A read-only port (projection/DTO, no mutation) uses `QueryInterface` to signal the read role at the call site.

Examples: `OrderRepositoryInterface` (mutated aggregate), `OrderSearchQueryInterface` and `OrderTimelineQueryInterface` (read).

## DI Configuration

```yaml
# config/services.yaml
services:
    # Bind ports to adapters
    App\User\Domain\Port\Out\UserRepositoryInterface:
        alias: App\User\Infrastructure\Adapter\Out\Doctrine\DoctrineUserRepository

    App\Payment\Domain\Port\Out\PaymentGatewayInterface:
        alias: App\Payment\Infrastructure\Adapter\Out\Payment\StripePaymentGateway

    App\Notification\Domain\Port\Out\NotificationServiceInterface:
        alias: App\Notification\Infrastructure\Adapter\Out\Storage\SymfonyMailerAdapter
```

## References

See `references/` for detailed guides:
- `port-adapter-examples.md` — Full examples for various port types
- `di-configuration.md` — Advanced DI configuration patterns

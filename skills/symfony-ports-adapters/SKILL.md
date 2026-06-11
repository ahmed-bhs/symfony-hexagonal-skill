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

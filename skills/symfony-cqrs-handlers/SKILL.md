---
description: "Symfony CQRS command/query handlers — commands, queries, handlers, bus configuration, use cases. Triggers on: command, query, handler, CQRS, bus, use case, command handler, query handler, message bus"
---

# Symfony CQRS Handlers

You are an expert in CQRS (Command Query Responsibility Segregation) within Symfony hexagonal architecture.

## When to Activate

- User wants to create a command or query
- User needs a handler for a use case
- User asks about CQRS patterns or message bus configuration
- User mentions "use case", "action", "operation" in application context

## Command Pattern

Commands represent write operations (create, update, delete). They are DTOs dispatched to the command bus.

### Rules
- `final readonly class` — immutable after construction
- Named as imperative verb use case: `RegisterUser`, `PlaceOrder`, `CancelSubscription`
- Message class and its handler are colocated in `Application/UseCase/Command/{UseCaseName}/`
- Message named `{UseCaseName}Command`, handler `{UseCaseName}Handler`
- Contains only primitive types and value objects — no entities
- Handler returns `void` or a scalar identifier (string ID)
- One handler per command

### Template
```php
namespace App\{Module}\Application\UseCase\Command\{UseCaseName};

final readonly class {UseCaseName}Command
{
    public function __construct(
        public string $param1,
        public string $param2,
        // only primitives and simple types
    ) {
    }
}
```

### Handler Template
The handler implements a neutral marker interface and receives the bus as a port;
the Messenger wiring lives in `services.yaml`, keeping `use Symfony\...` out of
Application.

```php
namespace App\{Module}\Application\UseCase\Command\{UseCaseName};

use App\{Module}\Domain\Port\Out\{Repository}Interface;
use App\Shared\Application\Bus\CommandHandlerInterface;

final readonly class {UseCaseName}Handler implements CommandHandlerInterface
{
    public function __construct(
        private {Repository}Interface $repository,
    ) {
    }

    public function __invoke({UseCaseName}Command $command): void
    {
        // 1. Reconstruct/create domain objects
        // 2. Execute business logic
        // 3. Persist via port
        // NO side-effects here — use domain events
    }
}
```

> **Alternative (pragmatic):** annotating the handler with
> `#[AsMessageHandler(bus: 'command.bus')]` is also valid — more explicit and
> idiomatic, at the cost of one inert framework attribute in Application. Pick one
> convention per project and keep it consistent.

## Query Pattern

Queries represent read operations. They return DTOs, never domain entities.

### Rules
- `final readonly class` — immutable
- Named descriptively: `GetUserById`, `ListActiveOrders`, `SearchProducts`
- Message class and its handler are colocated in `Application/UseCase/Query/{UseCaseName}/`
- Message named `{UseCaseName}Query`, handler `{UseCaseName}Handler`
- Handler MUST return a DTO or array of DTOs
- Handler NEVER modifies state
- May use read-optimized ports (separate from write ports)

### Template
```php
namespace App\{Module}\Application\UseCase\Query\{UseCaseName};

final readonly class {UseCaseName}Query
{
    public function __construct(
        public string $identifier,
        // filter/pagination params
    ) {
    }
}
```

### Handler Template
```php
namespace App\{Module}\Application\UseCase\Query\{UseCaseName};

use App\{Module}\Application\DTO\{Entity}DTO;
use App\{Module}\Domain\Port\Out\{Repository}Interface;
use App\Shared\Application\Bus\QueryHandlerInterface;

final readonly class {UseCaseName}Handler implements QueryHandlerInterface
{
    public function __construct(
        private {Repository}Interface $repository,
    ) {
    }

    public function __invoke({UseCaseName}Query $query): ?{Entity}DTO
    {
        $entity = $this->repository->findById($query->identifier);
        if ($entity === null) {
            return null;
        }

        return {Entity}DTO::fromEntity($entity);
    }
}
```

## DTO Pattern

```php
namespace App\{Module}\Application\DTO;

final readonly class {Entity}DTO
{
    public function __construct(
        public string $id,
        public string $field1,
        public string $field2,
    ) {
    }

    public static function fromEntity(/* entity */): self
    {
        return new self(
            id: (string) $entity->id(),
            field1: $entity->field1(),
            field2: $entity->field2(),
        );
    }
}
```

## References

See `references/` for detailed guides:
- `command-patterns.md` — Full command examples with validation
- `query-patterns.md` — Query patterns with pagination and filtering
- `bus-configuration.md` — Messenger bus setup and middleware

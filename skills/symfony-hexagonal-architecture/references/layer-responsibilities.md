# Layer Responsibilities

## Domain Layer

**Purpose**: Core business logic, rules, and invariants. Pure PHP only.

**Contains**:
- **Entities**: Aggregate roots and entities with business logic
- **Value Objects**: Immutable, self-validating objects (Email, Money, UserId)
- **Domain Events**: Immutable records of something that happened (past-tense naming)
- **Exceptions**: Domain-specific exception classes
- **Ports/Out**: Interfaces for driven dependencies the domain logic needs (repositories, domain calculators)

**Rules**:
- NO framework imports (`Symfony\`, `Doctrine\`, `Psr\`)
- NO infrastructure concerns (database, HTTP, filesystem)
- Entities use private constructors + static factory methods
- All state changes emit domain events
- Value objects are `final readonly` with self-validation in constructor

**Example Entity**:
```php
namespace App\User\Domain\Entity;

use App\User\Domain\Event\UserRegistered;
use App\User\Domain\ValueObject\Email;
use App\User\Domain\ValueObject\UserId;

final class User
{
    private array $domainEvents = [];

    private function __construct(
        private UserId $id,
        private Email $email,
        private string $name,
        private \DateTimeImmutable $createdAt,
    ) {
    }

    public static function register(UserId $id, Email $email, string $name): self
    {
        $user = new self($id, $email, $name, new \DateTimeImmutable());
        $user->recordEvent(new UserRegistered($id, $email));
        return $user;
    }

    public function pullDomainEvents(): array
    {
        $events = $this->domainEvents;
        $this->domainEvents = [];
        return $events;
    }

    private function recordEvent(object $event): void
    {
        $this->domainEvents[] = $event;
    }
}
```

## Application Layer

**Purpose**: Use case orchestration. Coordinates domain objects and ports.

**Contains** (use-case folders — one folder per capability):
- **Commands**: Write operation DTOs (final readonly), in `UseCase/Command/{Name}/`
- **Command Handlers**: Execute write operations via `__invoke`, colocated with their Command
- **Queries**: Read operation DTOs (final readonly), in `UseCase/Query/{Name}/`
- **Query Handlers**: Execute read operations, return DTOs, colocated with their Query
- **Ports/Out**: Technical driven ports (storage, read-model queries, token gen)
- **DTOs**: Data transfer objects for input/output
- **Event Handlers**: React to domain events (side-effects)

**Rules**:
- May depend on Domain layer only (and framework vendors used as orchestration tools)
- Handlers receive ports via constructor injection
- Command handlers return void or a scalar ID
- Query handlers return DTOs, never entities
- Side-effects (email, notifications) go in Event Handlers, not Command Handlers

**Handler binding** — prefer a framework-free Application: the handler implements
a neutral marker interface and receives the bus as a port, with the Messenger
wiring done in `services.yaml`. This keeps `use Symfony\...` out of Application.

```php
namespace App\User\Application\UseCase\Command\RegisterUser;

use App\User\Domain\Entity\User;
use App\User\Domain\Port\Out\UserRepositoryInterface;
use App\User\Domain\ValueObject\Email;
use App\User\Domain\ValueObject\UserId;
use App\Shared\Application\Bus\CommandHandlerInterface;
use App\Shared\Application\Port\In\EventBusInterface;

final readonly class RegisterUserHandler implements CommandHandlerInterface
{
    public function __construct(
        private UserRepositoryInterface $userRepository,
        private EventBusInterface $eventBus,
    ) {
    }

    public function __invoke(RegisterUserCommand $command): string
    {
        $userId = UserId::generate();
        $user = User::register($userId, new Email($command->email), $command->name);
        $this->userRepository->save($user);

        foreach ($user->pullDomainEvents() as $event) {
            $this->eventBus->publish($event);
        }

        return (string) $userId;
    }
}
```

> **Alternative (pragmatic):** annotating the handler with
> `#[AsMessageHandler(bus: 'command.bus')]` is also valid. It is more explicit
> and idiomatic, at the cost of one inert framework attribute in Application.
> Both work; pick one convention per project and keep it consistent.

## Infrastructure Layer (Adapters)

**Purpose**: Technical implementations on both sides of the hexagon. Split by
flow direction under `Infrastructure/Adapter/{In,Out}/`.

**Driven adapters (`Adapter/Out/`)** — implement ports the app calls:
- **Doctrine**: repositories implementing port interfaces, ORM mappings (`Mapping/`)
- **Storage**: filesystem, S3, mailer adapters
- **Bus**: Messenger bus implementations of the driving-port interfaces

**Driving adapters (`Adapter/In/`)** — invoke the app from outside:
- **ApiPlatform / CLI / Admin**: entry points that dispatch commands/queries
- **Security / Validation**: voters, user checkers, input constraints

**Rules**:
- Driven adapters implement Domain/Application port interfaces
- Contains ALL framework-specific code (Doctrine, HTTP clients)
- ORM mapping may use attributes on the Domain entity (inert metadata) or be externalised here as XML — either is fine; DB-access validators (`UniqueEntity`) always live here
- Repository adapters translate between Domain entities and Doctrine

**Example Driven Adapter (repository)**:
```php
namespace App\User\Infrastructure\Adapter\Out\Doctrine;

use App\User\Domain\Entity\User;
use App\User\Domain\Port\Out\UserRepositoryInterface;
use App\User\Domain\ValueObject\UserId;
use Doctrine\ORM\EntityManagerInterface;

final readonly class DoctrineUserRepository implements UserRepositoryInterface
{
    public function __construct(
        private EntityManagerInterface $entityManager,
    ) {
    }

    public function save(User $user): void
    {
        $this->entityManager->persist($user);
        $this->entityManager->flush();
    }

    public function findById(UserId $id): ?User
    {
        return $this->entityManager->find(User::class, (string) $id);
    }
}
```

## Driving Adapters (`Adapter/In/`)

**Purpose**: Deliver the application to the outside world (HTTP, CLI, API
Platform, EasyAdmin). These are adapters too — there is no separate Presentation
layer.

**Contains**:
- **ApiPlatform**: REST resources, processors, providers (thin — delegate to bus)
- **CLI**: Symfony Console commands
- **Admin**: EasyAdmin CRUD controllers, LiveComponents

**Rules**:
- Entry points are thin — dispatch commands/queries via the bus port only
- Input validation happens here (request validation, constraints)
- Response formatting uses standard JSON payload
- Every endpoint must have `#[IsGranted]` or Voter check
- No business logic in driving adapters

**Example Driving Adapter (controller)**:
```php
namespace App\User\Infrastructure\Adapter\In\ApiPlatform\RegisterUser;

use App\User\Application\UseCase\Command\RegisterUser\RegisterUserCommand;
use App\Shared\Application\Port\In\CommandBusInterface;
use App\Shared\Infrastructure\Adapter\In\Http\ApiResponseTrait;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Routing\Attribute\Route;
use Symfony\Component\Security\Http\Attribute\IsGranted;

#[Route('/api/users')]
final class RegisterUserController
{
    use ApiResponseTrait;

    public function __construct(
        private readonly CommandBusInterface $commandBus,
    ) {
    }

    #[Route('', methods: ['POST'])]
    #[IsGranted('ROLE_USER_CREATE')]
    public function __invoke(Request $request): JsonResponse
    {
        $data = json_decode($request->getContent(), true);
        $command = new RegisterUserCommand($data['email'], $data['name']);

        $userId = $this->commandBus->dispatch($command);

        return $this->success(['id' => $userId], 201);
    }
}
```

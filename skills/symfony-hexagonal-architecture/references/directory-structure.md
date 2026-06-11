# Directory Structure Reference

## Full Project Layout

Module-first layout. Each bounded context owns its three layers. Entry points
(controllers, CLI, API Platform, EasyAdmin) are **driving adapters** under
`Infrastructure/Adapter/In/`; persistence, storage, buses and external clients
are **driven adapters** under `Infrastructure/Adapter/Out/`. There is no separate
`Presentation/` layer — everything that touches the outside world is an adapter,
only the direction differs.

```
project-root/
├── config/
│   ├── packages/
│   │   ├── messenger.yaml
│   │   ├── security.yaml
│   │   └── doctrine.yaml
│   ├── services.yaml
│   └── routes/
├── src/
│   └── {Module}/
│       ├── Domain/
│       │   ├── Entity/
│       │   │   └── {AggregateRoot}.php
│       │   ├── ValueObject/
│       │   │   └── {ValueObject}.php
│       │   ├── Event/
│       │   │   └── {PastTenseEvent}.php
│       │   ├── Exception/
│       │   │   └── {DomainException}.php
│       │   └── Port/
│       │       └── Out/
│       │           └── {RepositoryInterface}.php
│       ├── Application/
│       │   ├── UseCase/
│       │   │   ├── Command/
│       │   │   │   └── {UseCaseName}/
│       │   │   │       ├── {UseCaseName}Command.php
│       │   │   │       └── {UseCaseName}Handler.php
│       │   │   └── Query/
│       │   │       └── {UseCaseName}/
│       │   │           ├── {UseCaseName}Query.php
│       │   │           └── {UseCaseName}Handler.php
│       │   ├── Port/
│       │   │   └── Out/
│       │   │       └── {TechnicalPortInterface}.php
│       │   ├── DTO/
│       │   │   └── {DataDTO}.php
│       │   └── EventHandler/
│       │       └── {WhenEventHandler}.php
│       └── Infrastructure/
│           └── Adapter/
│               ├── In/
│               │   ├── ApiPlatform/
│               │   │   └── {UseCase}/
│               │   ├── CLI/
│               │   │   └── {ActionCommand}.php
│               │   ├── Admin/
│               │   │   └── {Entity}CrudController.php
│               │   ├── Security/
│               │   │   └── {Voter}.php
│               │   └── Validation/
│               │       └── {Constraint}Validator.php
│               └── Out/
│                   ├── Doctrine/
│                   │   ├── Doctrine{Entity}Repository.php
│                   │   └── Mapping/
│                   │       └── {Entity}.orm.xml
│                   ├── Storage/
│                   │   └── {StorageAdapter}.php
│                   └── Bus/
│                       └── {MessengerBus}.php
├── tests/
│   ├── Unit/
│   │   └── {Module}/Domain/
│   ├── Integration/
│   │   └── {Module}/Infrastructure/
│   └── Behat/
└── migrations/
```

## Shared Module

Cross-cutting contracts live in a `Shared/` module. The command/query/event
**buses are the driving ports** of the whole system and live under
`Shared/Application/Port/In/`. Their Messenger implementations are driven
adapters under `Shared/Infrastructure/Adapter/Out/Bus/`.

```
src/Shared/
├── Domain/
│   ├── Contract/{CrossCuttingInterface}.php
│   ├── Event/DomainEventInterface.php
│   ├── Trait/{Timestampable,EventRecorder}Trait.php
│   └── ValueObject/AggregateRootId.php
├── Application/
│   ├── Port/In/
│   │   ├── CommandBusInterface.php
│   │   ├── QueryBusInterface.php
│   │   └── EventBusInterface.php
│   └── Bus/
│       ├── CommandHandlerInterface.php   # marker, not a port
│       └── QueryHandlerInterface.php
└── Infrastructure/
    └── Adapter/
        ├── In/Admin/{DashboardController,SecurityController}.php
        └── Out/
            ├── Bus/{MessengerCommandBus,MessengerQueryBus,MessengerEventBus}.php
            └── Doctrine/{DoctrineFilter}.php
```

## Example: User Module

```
src/User/
├── Domain/
│   ├── Entity/User.php
│   ├── ValueObject/Email.php
│   ├── ValueObject/UserId.php
│   ├── Event/UserRegistered.php
│   ├── Exception/UserAlreadyExistsException.php
│   └── Port/Out/UserRepositoryInterface.php
├── Application/
│   ├── UseCase/
│   │   ├── Command/RegisterUser/
│   │   │   ├── RegisterUserCommand.php
│   │   │   └── RegisterUserHandler.php
│   │   └── Query/GetUserById/
│   │       ├── GetUserByIdQuery.php
│   │       └── GetUserByIdHandler.php
│   ├── Port/Out/PasswordHasherInterface.php
│   ├── DTO/UserDTO.php
│   └── EventHandler/WhenUserRegisteredSendWelcomeEmail.php
└── Infrastructure/
    └── Adapter/
        ├── In/
        │   ├── ApiPlatform/RegisterUser/RegisterUserProcessor.php
        │   ├── Admin/UserCrudController.php
        │   └── Security/UserVoter.php
        └── Out/
            ├── Doctrine/
            │   ├── DoctrineUserRepository.php
            │   └── Mapping/User.orm.xml
            └── Storage/MailerAdapter.php
```

## Namespace Convention (module-first)

```php
// Domain
namespace App\User\Domain\Entity;
namespace App\User\Domain\ValueObject;
namespace App\User\Domain\Event;
namespace App\User\Domain\Exception;
namespace App\User\Domain\Port\Out;

// Application
namespace App\User\Application\UseCase\Command\RegisterUser;
namespace App\User\Application\UseCase\Query\GetUserById;
namespace App\User\Application\Port\Out;
namespace App\User\Application\DTO;
namespace App\User\Application\EventHandler;

// Infrastructure — driving adapters (In)
namespace App\User\Infrastructure\Adapter\In\ApiPlatform\RegisterUser;
namespace App\User\Infrastructure\Adapter\In\Admin;
namespace App\User\Infrastructure\Adapter\In\Security;

// Infrastructure — driven adapters (Out)
namespace App\User\Infrastructure\Adapter\Out\Doctrine;
namespace App\User\Infrastructure\Adapter\Out\Storage;
```

## Port placement

- **Domain ports** (`Domain/Port/Out/`): contracts the domain logic itself needs
  — repositories used by entities/domain services, domain calculators.
- **Application ports** (`Application/Port/Out/`): orchestration concerns the
  domain ignores — storage, token generators, read-model queries, mailers.
- **Driving ports** (`Shared/Application/Port/In/`): the buses through which
  driving adapters invoke use cases. No per-use-case `Port/In` interface is
  required — the `Command`/`Query` message plus the bus interface form the entry
  contract.

## services.yaml Module Binding

```yaml
services:
    App\:
        resource: '../src/'

    # Driving ports -> Messenger bus adapters
    App\Shared\Application\Port\In\CommandBusInterface:
        alias: App\Shared\Infrastructure\Adapter\Out\Bus\MessengerCommandBus

    # Bind a domain port to its Doctrine adapter
    App\User\Domain\Port\Out\UserRepositoryInterface:
        alias: App\User\Infrastructure\Adapter\Out\Doctrine\DoctrineUserRepository
```

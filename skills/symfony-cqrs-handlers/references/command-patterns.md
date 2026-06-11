# Command Patterns

## Full Command + Handler Example

Each use case is a folder under `Application/UseCase/Command/{UseCaseName}/` holding
the message and its handler colocated.

### Command

```php
namespace App\User\Application\UseCase\Command\RegisterUser;

use Symfony\Component\Validator\Constraints as Assert;

final readonly class RegisterUserCommand
{
    public function __construct(
        #[Assert\NotBlank]
        #[Assert\Email]
        public string $email,

        #[Assert\NotBlank]
        #[Assert\Length(min: 2, max: 100)]
        public string $name,

        #[Assert\NotBlank]
        #[Assert\Length(min: 8)]
        public string $password,
    ) {
    }
}
```

### Handler

The handler implements the neutral `CommandHandlerInterface` marker; Messenger
wiring lives in `services.yaml`.

```php
namespace App\User\Application\UseCase\Command\RegisterUser;

use App\User\Domain\Entity\User;
use App\User\Domain\Exception\UserAlreadyExistsException;
use App\User\Domain\Port\Out\UserRepositoryInterface;
use App\User\Domain\ValueObject\Email;
use App\User\Domain\ValueObject\UserId;
use App\User\Application\Port\Out\PasswordHasherInterface;
use App\Shared\Application\Bus\CommandHandlerInterface;

final readonly class RegisterUserHandler implements CommandHandlerInterface
{
    public function __construct(
        private UserRepositoryInterface $userRepository,
        private PasswordHasherInterface $passwordHasher,
    ) {
    }

    public function __invoke(RegisterUserCommand $command): string
    {
        $email = new Email($command->email);

        if ($this->userRepository->existsByEmail($email)) {
            throw UserAlreadyExistsException::withEmail($email);
        }

        $userId = UserId::generate();
        $hashedPassword = $this->passwordHasher->hashPassword(/* ... */);
        $user = User::register($userId, $email, $command->name, $hashedPassword);

        $this->userRepository->save($user);

        return $userId->value;
    }
}
```

> **Alternative (pragmatic):** annotating the handler with
> `#[AsMessageHandler(bus: 'command.bus')]` is also valid — more explicit, at the
> cost of one inert framework attribute in Application. Pick one convention per
> project and keep it consistent.

## Command That Updates

`Application/UseCase/Command/UpdateUserProfile/`:

```php
final readonly class UpdateUserProfileCommand
{
    public function __construct(
        public string $userId,
        public ?string $name = null,
        public ?string $bio = null,
    ) {
    }
}

final readonly class UpdateUserProfileHandler implements CommandHandlerInterface
{
    public function __construct(
        private UserRepositoryInterface $userRepository,
    ) {
    }

    public function __invoke(UpdateUserProfileCommand $command): void
    {
        $user = $this->userRepository->findById(new UserId($command->userId));
        if ($user === null) {
            throw UserNotFoundException::withId($command->userId);
        }

        if ($command->name !== null) {
            $user->changeName($command->name);
        }
        if ($command->bio !== null) {
            $user->updateBio($command->bio);
        }

        $this->userRepository->save($user);
    }
}
```

## Command That Deletes

`Application/UseCase/Command/DeactivateUser/`:

```php
final readonly class DeactivateUserCommand
{
    public function __construct(
        public string $userId,
        public string $reason,
    ) {
    }
}

final readonly class DeactivateUserHandler implements CommandHandlerInterface
{
    public function __construct(
        private UserRepositoryInterface $userRepository,
    ) {
    }

    public function __invoke(DeactivateUserCommand $command): void
    {
        $user = $this->userRepository->findById(new UserId($command->userId));
        if ($user === null) {
            throw UserNotFoundException::withId($command->userId);
        }

        $user->deactivate($command->reason);
        $this->userRepository->save($user);
    }
}
```

## Dispatching Commands from Driving Adapters

Driving adapters (controllers, CLI, API Platform) dispatch through the
`CommandBusInterface` driving port — never the raw Messenger bus. The bus adapter
unwraps the handler result so the port returns it directly.

```php
use App\Shared\Application\Port\In\CommandBusInterface;

// In a driving adapter:
$userId = $this->commandBus->dispatch(
    new RegisterUserCommand($email, $name, $password),
);
```

## Async Commands

Some commands can be dispatched asynchronously:

```yaml
# messenger.yaml
framework:
    messenger:
        routing:
            'App\Report\Application\UseCase\Command\GenerateReport\GenerateReportCommand': async
            'App\Email\Application\UseCase\Command\SendBulkEmail\SendBulkEmailCommand': async
```

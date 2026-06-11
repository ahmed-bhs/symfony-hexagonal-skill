---
description: "Symfony testing — PHPUnit test organization, unit tests, integration tests, functional tests, TDD workflow, PHPStan, mocking ports, test patterns. Triggers on: test, PHPUnit, TDD, unit test, integration test, functional test, mock, testing, test driven, code quality, PHPStan"
---

# Symfony Testing

You are an expert in testing within Symfony hexagonal architecture.

## When to Activate

- User needs to write tests
- User asks about test organization or TDD
- User needs mocking strategies for ports
- User asks about PHPStan or code quality
- User mentions test-driven development

## Test Directory Structure (Mirrors src/)

```
tests/                                  # mirrors src/ module-first
├── Unit/
│   └── {Module}/
│       ├── Domain/
│       │   ├── Entity/{Entity}Test.php
│       │   └── ValueObject/{ValueObject}Test.php
│       └── Application/                 # handlers with mocked ports
│           └── UseCase/Command/{Name}/{Name}HandlerTest.php
├── Integration/
│   └── {Module}/
│       └── Infrastructure/             # real DB — adapter ⇄ port contract
│           └── Adapter/Out/Doctrine/{Repository}Test.php
├── Functional/                         # full HTTP stack (driving adapters)
│   └── {Module}/
│       └── Infrastructure/Adapter/In/{Controller}Test.php
├── Behat/                              # E2E business scenarios (Given/When/Then)
└── Support/
    └── Double/
        └── InMemory{Repository}.php
```

> The test tree mirrors `src/` module-first. Domain unit tests are pure PHP;
> handler unit tests mock the ports; integration tests hit a real DB (same engine
> as prod); Functional drives the HTTP stack; Behat covers critical end-to-end
> journeys in business language.

## Test Types by Layer

### Unit Tests (Domain) — Pure PHP, no framework
```php
namespace Tests\Unit\User\Domain\ValueObject;

use App\User\Domain\ValueObject\Email;
use App\User\Domain\Exception\InvalidEmailException;
use PHPUnit\Framework\TestCase;

final class EmailTest extends TestCase
{
    public function test_valid_email_is_created(): void
    {
        $email = new Email('user@example.com');
        $this->assertSame('user@example.com', $email->value);
    }

    public function test_invalid_email_throws_exception(): void
    {
        $this->expectException(InvalidEmailException::class);
        new Email('invalid');
    }

    public function test_emails_are_equal(): void
    {
        $email1 = new Email('user@example.com');
        $email2 = new Email('user@example.com');
        $this->assertTrue($email1->equals($email2));
    }
}
```

### Handler Unit Tests (Application) — Mock ports
```php
namespace Tests\Unit\User\Application\UseCase\Command\RegisterUser;

use App\User\Application\UseCase\Command\RegisterUser\RegisterUserCommand;
use App\User\Application\UseCase\Command\RegisterUser\RegisterUserHandler;
use App\User\Domain\Port\Out\UserRepositoryInterface;
use App\Shared\Application\Port\In\EventBusInterface;
use PHPUnit\Framework\TestCase;

final class RegisterUserHandlerTest extends TestCase
{
    public function test_it_registers_a_user(): void
    {
        $repository = $this->createMock(UserRepositoryInterface::class);
        $repository->expects($this->once())->method('save');
        $repository->method('existsByEmail')->willReturn(false);

        $handler = new RegisterUserHandler($repository, $this->createMock(EventBusInterface::class));
        $result = $handler(new RegisterUserCommand('test@example.com', 'Test', 'password123'));

        $this->assertNotEmpty($result);
    }
}
```

> Mock only the **ports** (interfaces). Never mock internal domain collaborators.

### Functional Tests (driving adapter) — Full HTTP stack
```php
namespace Tests\Functional\User\Infrastructure\Adapter\In;

use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

final class UserControllerTest extends WebTestCase
{
    public function test_create_user(): void
    {
        $client = static::createClient();
        $client->request('POST', '/api/users', [], [], [
            'CONTENT_TYPE' => 'application/json',
        ], json_encode([
            'email' => 'test@example.com',
            'name' => 'Test User',
            'password' => 'Password123',
        ]));

        $this->assertResponseStatusCodeSame(201);
        $response = json_decode($client->getResponse()->getContent(), true);
        $this->assertNotNull($response['result']['id']);
        $this->assertNull($response['error']);
    }
}
```

## TDD Workflow

1. Write test first (RED)
2. Write minimal code to pass (GREEN)
3. Refactor (REFACTOR)
4. Run PHPStan
5. Repeat

## References

See `references/` for detailed guides:
- `test-organization.md` — phpunit.xml config, test suites, fixtures
- `test-patterns.md` — In-memory adapters, test builders, assertion helpers

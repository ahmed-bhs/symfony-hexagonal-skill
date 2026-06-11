# Dependency Injection Configuration

## Basic Port-to-Adapter Binding

```yaml
# config/services.yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true
        bind:
            $commandBus: '@command.bus'
            $queryBus: '@query.bus'
            $eventBus: '@event.bus'

    # Auto-register all services
    App\:
        resource: '../src/'
        exclude:
            - '../src/*/Domain/Entity/'
            - '../src/*/Domain/ValueObject/'
            - '../src/*/Domain/Event/'
            - '../src/*/Domain/Exception/'
            - '../src/Kernel.php'

    # Port → Adapter bindings
    App\User\Domain\Port\Out\UserRepositoryInterface:
        alias: App\User\Infrastructure\Adapter\Out\Doctrine\DoctrineUserRepository

    App\Payment\Domain\Port\Out\PaymentGatewayInterface:
        alias: App\Payment\Infrastructure\Adapter\Out\Payment\StripePaymentGateway

    App\Notification\Domain\Port\Out\NotificationServiceInterface:
        alias: App\Notification\Infrastructure\Adapter\Out\Storage\SymfonyMailerAdapter
```

## Environment-Specific Adapters

Use different adapters per environment:

```yaml
# config/services.yaml (production default)
services:
    App\Notification\Domain\Port\Out\NotificationServiceInterface:
        alias: App\Notification\Infrastructure\Adapter\Out\Storage\SymfonyMailerAdapter

# config/services_dev.yaml
services:
    App\Notification\Domain\Port\Out\NotificationServiceInterface:
        alias: App\Notification\Infrastructure\Adapter\Out\Storage\NullNotificationService

# config/services_test.yaml
services:
    App\User\Domain\Port\Out\UserRepositoryInterface:
        alias: Tests\Support\Adapter\InMemoryUserRepository
```

## Adapter with Configuration

```yaml
services:
    App\Payment\Infrastructure\Adapter\Out\Payment\StripePaymentGateway:
        arguments:
            $stripe: '@stripe.client'

    stripe.client:
        class: Stripe\StripeClient
        arguments:
            - '%env(STRIPE_SECRET_KEY)%'

    App\Notification\Infrastructure\Adapter\Out\Storage\SymfonyMailerAdapter:
        arguments:
            $fromEmail: '%env(MAILER_FROM)%'

    App\Document\Infrastructure\Adapter\Out\Storage\S3FileStorage:
        arguments:
            $bucket: '%env(AWS_S3_BUCKET)%'
```

## Tagged Services (Multiple Adapters)

When you need multiple implementations of the same port:

```yaml
services:
    App\Notification\Infrastructure\Adapter\Out\Storage\EmailNotificationAdapter:
        tags: ['app.notification_channel']

    App\Notification\Infrastructure\Adapter\Out\Storage\SmsNotificationAdapter:
        tags: ['app.notification_channel']

    App\Notification\Infrastructure\Adapter\Out\Storage\CompositeNotificationService:
        arguments:
            $channels: !tagged_iterator 'app.notification_channel'
```

## Decorator Pattern

```yaml
services:
    App\User\Infrastructure\Adapter\Out\Doctrine\DoctrineUserRepository: ~

    App\User\Infrastructure\Adapter\Out\Doctrine\CachedUserRepository:
        decorates: App\User\Infrastructure\Adapter\Out\Doctrine\DoctrineUserRepository
        arguments:
            $inner: '@.inner'
            $cache: '@cache.app'
```

```php
final readonly class CachedUserRepository implements UserRepositoryInterface
{
    public function __construct(
        private UserRepositoryInterface $inner,
        private CacheItemPoolInterface $cache,
    ) {
    }

    public function findById(UserId $id): ?User
    {
        $cacheKey = 'user_' . $id->value;
        $item = $this->cache->getItem($cacheKey);

        if ($item->isHit()) {
            return $item->get();
        }

        $user = $this->inner->findById($id);
        if ($user !== null) {
            $item->set($user)->expiresAfter(3600);
            $this->cache->save($item);
        }

        return $user;
    }

    // delegate other methods to $this->inner
}
```

## Autowiring Interfaces

For interfaces that have only one implementation, Symfony can autowire automatically:

```yaml
services:
    # If there's only one class implementing UserRepositoryInterface,
    # Symfony will autowire it automatically.
    # Explicit alias is still recommended for clarity:
    App\User\Domain\Port\Out\UserRepositoryInterface: '@App\User\Infrastructure\Adapter\Out\Doctrine\DoctrineUserRepository'
```

## Module-Scoped services.yaml

For large projects, split configuration per module:

```yaml
# config/services.yaml
imports:
    - { resource: 'services/user.yaml' }
    - { resource: 'services/order.yaml' }
    - { resource: 'services/payment.yaml' }
```

```yaml
# config/services/user.yaml
services:
    App\User\Domain\Port\Out\UserRepositoryInterface:
        alias: App\User\Infrastructure\Adapter\Out\Doctrine\DoctrineUserRepository

    App\User\Domain\Port\Out\UserReadRepositoryInterface:
        alias: App\User\Infrastructure\Adapter\Out\Doctrine\DbalUserReadRepository
```

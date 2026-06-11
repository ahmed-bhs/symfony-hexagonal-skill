# Domain Event Patterns

## Rules
- Past-tense naming: something that already happened
- `final readonly class` — immutable after creation
- Carry IDs and scalar data, not entity references
- Include `occurredAt` timestamp
- Created inside entities on state changes
- Dispatched by infrastructure (repository adapter) after persistence

## Event Naming Convention

| Action | Event Name |
|--------|-----------|
| User registers | `UserRegistered` |
| Order is placed | `OrderPlaced` |
| Payment fails | `PaymentFailed` |
| Product price changes | `ProductPriceChanged` |
| Account is suspended | `AccountSuspended` |

## Base Event Interface (Shared Kernel)

```php
namespace App\Shared\Domain\Event;

interface DomainEvent
{
    public function occurredAt(): \DateTimeImmutable;
    public function aggregateId(): string;
}
```

## Event Template

```php
namespace App\{Module}\Domain\Event;

use App\Shared\Domain\Event\DomainEvent;

final readonly class {EntityAction} implements DomainEvent
{
    public function __construct(
        private string $entityId,
        // additional data relevant to the event
        private \DateTimeImmutable $occurredAt = new \DateTimeImmutable(),
    ) {
    }

    public function aggregateId(): string
    {
        return $this->entityId;
    }

    public function occurredAt(): \DateTimeImmutable
    {
        return $this->occurredAt;
    }
}
```

## Event Handler (Application Layer)

Event handlers live in `Application/{Module}/EventHandler/` and handle side-effects.
Like command/query handlers, they implement a neutral marker interface; the
Messenger wiring to `event.bus` lives in `services.yaml`.

```php
namespace App\User\Application\EventHandler;

use App\User\Domain\Event\UserRegistered;
use App\Notification\Domain\Port\Out\NotificationServiceInterface;
use App\Shared\Application\Bus\EventHandlerInterface;

final readonly class WhenUserRegisteredSendWelcomeEmail implements EventHandlerInterface
{
    public function __construct(
        private NotificationServiceInterface $notificationService,
    ) {
    }

    public function __invoke(UserRegistered $event): void
    {
        $this->notificationService->sendWelcomeEmail($event->userId, $event->email);
    }
}
```

> **Alternative (pragmatic):** annotating the handler with
> `#[AsMessageHandler(bus: 'event.bus')]` is also valid — more explicit, at the
> cost of one inert framework attribute in Application. Pick one convention per
> project and keep it consistent.

## Event Handler Naming Convention

Use `When{Event}{Action}` pattern:
- `WhenUserRegisteredSendWelcomeEmail`
- `WhenOrderPlacedReserveInventory`
- `WhenPaymentFailedNotifyCustomer`
- `WhenProductPriceChangedUpdateCatalog`

## Event Dispatch Flow

```
1. Entity.doAction() → records event internally
2. Repository.save(entity) → persists to DB
3. Repository pulls events from entity → dispatches to event bus
4. Event bus → routes to registered handlers
5. Handlers execute side-effects
```

## Multiple Handlers per Event

One event can trigger multiple handlers. Each handler should be independent:

```php
// Handler 1: Send email
final readonly class WhenOrderPlacedSendConfirmation implements EventHandlerInterface { ... }

// Handler 2: Reserve inventory
final readonly class WhenOrderPlacedReserveInventory implements EventHandlerInterface { ... }

// Handler 3: Update analytics
final readonly class WhenOrderPlacedTrackAnalytics implements EventHandlerInterface { ... }
```

## Cross-Module Events

When Module A needs to react to Module B's events, the handler lives in Module A's Application layer:

```php
// Order module reacts to Payment module's event
namespace App\Order\Application\EventHandler;

use App\Payment\Domain\Event\PaymentReceived;
use App\Order\Domain\Port\Out\OrderRepositoryInterface;
use App\Shared\Application\Bus\EventHandlerInterface;

final readonly class WhenPaymentReceivedConfirmOrder implements EventHandlerInterface
{
    public function __construct(
        private OrderRepositoryInterface $orderRepository,
    ) {
    }

    public function __invoke(PaymentReceived $event): void
    {
        $order = $this->orderRepository->findById($event->orderId);
        $order?->confirm();
        if ($order !== null) {
            $this->orderRepository->save($order);
        }
    }
}
```

## Messenger Bus Configuration for Events

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        buses:
            command.bus:
                middleware:
                    - doctrine_transaction
            query.bus: ~
            event.bus:
                default_middleware: allow_no_handlers
```

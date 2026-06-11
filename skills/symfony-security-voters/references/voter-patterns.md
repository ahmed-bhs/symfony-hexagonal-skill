# Voter Patterns

## Where Voters Live

Voters are Infrastructure concerns — they live in `Infrastructure/Adapter/In/Security/`:

```
src/Order/Infrastructure/Adapter/In/Security/OrderVoter.php
src/Document/Infrastructure/Adapter/In/Security/DocumentVoter.php
```

## Full Voter Example: Order

```php
namespace App\Order\Infrastructure\Adapter\In\Security;

use App\Order\Domain\Entity\Order;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\Authorization\Voter\Voter;
use Symfony\Component\Security\Core\User\UserInterface;

final class OrderVoter extends Voter
{
    public const VIEW = 'ORDER_VIEW';
    public const EDIT = 'ORDER_EDIT';
    public const CANCEL = 'ORDER_CANCEL';

    protected function supports(string $attribute, mixed $subject): bool
    {
        return in_array($attribute, [self::VIEW, self::EDIT, self::CANCEL])
            && $subject instanceof Order;
    }

    protected function voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token): bool
    {
        $user = $token->getUser();
        if (!$user instanceof UserInterface) {
            return false;
        }

        /** @var Order $order */
        $order = $subject;

        return match ($attribute) {
            self::VIEW => $this->canView($order, $user),
            self::EDIT => $this->canEdit($order, $user),
            self::CANCEL => $this->canCancel($order, $user),
            default => false,
        };
    }

    private function canView(Order $order, UserInterface $user): bool
    {
        // Owner can always view their orders
        if ($order->customerId() === $user->getUserIdentifier()) {
            return true;
        }

        // Staff with ORDER_VIEW role can view any order
        return in_array('ROLE_ORDER_VIEW', $user->getRoles(), true);
    }

    private function canEdit(Order $order, UserInterface $user): bool
    {
        // Only owner can edit, and only if order is pending
        return $order->customerId() === $user->getUserIdentifier()
            && $order->status()->value === 'pending';
    }

    private function canCancel(Order $order, UserInterface $user): bool
    {
        // Owner can cancel pending orders
        if ($order->customerId() === $user->getUserIdentifier() && $order->status()->value === 'pending') {
            return true;
        }

        // Admin can cancel any non-delivered order
        return in_array('ROLE_ADMIN', $user->getRoles(), true)
            && $order->status()->value !== 'delivered';
    }
}
```

## Controller Usage

```php
#[Route('/{id}', methods: ['PUT'])]
public function update(string $id, Request $request): JsonResponse
{
    $order = $this->queryBus->dispatch(new GetOrderByIdQuery($id));

    if ($order === null) {
        return $this->error('Order not found', 404);
    }

    // Voter checks authorization
    $this->denyAccessUnlessGranted(OrderVoter::EDIT, $order);

    // Proceed with update
    $this->commandBus->dispatch(new UpdateOrderCommand($id, /* ... */));

    return $this->noContent();
}
```

## Testing Voters

```php
namespace Tests\Integration\Order\Infrastructure\Security;

use App\Order\Domain\Entity\Order;
use App\Order\Infrastructure\Adapter\In\Security\OrderVoter;
use PHPUnit\Framework\TestCase;
use Symfony\Component\Security\Core\Authentication\Token\UsernamePasswordToken;
use Symfony\Component\Security\Core\Authorization\Voter\VoterInterface;

final class OrderVoterTest extends TestCase
{
    private OrderVoter $voter;

    protected function setUp(): void
    {
        $this->voter = new OrderVoter();
    }

    public function test_owner_can_view_their_order(): void
    {
        $order = $this->createOrder('user-123');
        $token = $this->createToken('user-123', ['ROLE_USER']);

        $result = $this->voter->vote($token, $order, [OrderVoter::VIEW]);

        $this->assertSame(VoterInterface::ACCESS_GRANTED, $result);
    }

    public function test_other_user_cannot_edit_order(): void
    {
        $order = $this->createOrder('user-123');
        $token = $this->createToken('user-456', ['ROLE_USER']);

        $result = $this->voter->vote($token, $order, [OrderVoter::EDIT]);

        $this->assertSame(VoterInterface::ACCESS_DENIED, $result);
    }

    private function createToken(string $userId, array $roles): UsernamePasswordToken
    {
        $user = $this->createMock(UserInterface::class);
        $user->method('getUserIdentifier')->willReturn($userId);
        $user->method('getRoles')->willReturn($roles);

        return new UsernamePasswordToken($user, 'main', $roles);
    }

    private function createOrder(string $customerId): Order
    {
        // Use reflection or factory to create test order
    }
}
```

## API Key / Service Authentication

For machine-to-machine APIs, combine with custom authenticators:

```php
#[Route('/api/webhooks/payment', methods: ['POST'])]
#[IsGranted('ROLE_WEBHOOK')]
public function handlePaymentWebhook(Request $request): JsonResponse
{
    // Authenticated via API key authenticator
    // Authorized via ROLE_WEBHOOK
}
```

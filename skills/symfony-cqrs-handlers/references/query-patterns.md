# Query Patterns

## Simple Query

`Application/UseCase/Query/GetUserById/` holds the query message and handler,
colocated. The handler implements the neutral `QueryHandlerInterface` marker.

```php
namespace App\User\Application\UseCase\Query\GetUserById;

final readonly class GetUserByIdQuery
{
    public function __construct(
        public string $userId,
    ) {
    }
}

final readonly class GetUserByIdHandler implements QueryHandlerInterface
{
    public function __construct(
        private UserRepositoryInterface $userRepository,
    ) {
    }

    public function __invoke(GetUserByIdQuery $query): ?UserDTO
    {
        $user = $this->userRepository->findById(new UserId($query->userId));
        if ($user === null) {
            return null;
        }

        return UserDTO::fromEntity($user);
    }
}
```

> **Alternative (pragmatic):** annotating the handler with
> `#[AsMessageHandler(bus: 'query.bus')]` is also valid — more explicit, at the
> cost of one inert framework attribute in Application. Pick one convention per
> project and keep it consistent.

## List Query with Pagination

`Application/UseCase/Query/ListUsers/`:

```php
final readonly class ListUsersQuery
{
    public function __construct(
        public int $page = 1,
        public int $limit = 20,
        public ?string $status = null,
        public ?string $search = null,
    ) {
    }
}

final readonly class ListUsersHandler implements QueryHandlerInterface
{
    public function __construct(
        private UserRepositoryInterface $userRepository,
    ) {
    }

    public function __invoke(ListUsersQuery $query): PaginatedResult
    {
        return $this->userRepository->findPaginated(
            page: $query->page,
            limit: $query->limit,
            status: $query->status,
            search: $query->search,
        );
    }
}
```

## Paginated Result DTO

```php
namespace App\Shared\Application\DTO;

final readonly class PaginatedResult
{
    public function __construct(
        public array $items,
        public int $total,
        public int $page,
        public int $limit,
    ) {
    }

    public function totalPages(): int
    {
        return (int) ceil($this->total / $this->limit);
    }

    public function hasNextPage(): bool
    {
        return $this->page < $this->totalPages();
    }
}
```

## Read-Optimized Port

For complex queries, define a separate read port that may use an optimized read-model:

```php
namespace App\Order\Domain\Port\Out;

use App\Shared\Application\DTO\PaginatedResult;

interface OrderReadRepositoryInterface
{
    public function findOrderSummaries(
        string $customerId,
        int $page,
        int $limit,
    ): PaginatedResult;

    public function findDashboardStats(string $customerId): OrderStatsDTO;
}
```

The adapter can use Doctrine DBAL directly for optimized reads:

```php
namespace App\Order\Infrastructure\Adapter\Out\Doctrine;

use App\Order\Domain\Port\Out\OrderReadRepositoryInterface;
use Doctrine\DBAL\Connection;

final readonly class DbalOrderReadRepository implements OrderReadRepositoryInterface
{
    public function __construct(
        private Connection $connection,
    ) {
    }

    public function findOrderSummaries(string $customerId, int $page, int $limit): PaginatedResult
    {
        $offset = ($page - 1) * $limit;

        $sql = 'SELECT id, status, total_amount, created_at FROM orders WHERE customer_id = :customerId ORDER BY created_at DESC LIMIT :limit OFFSET :offset';

        $rows = $this->connection->fetchAllAssociative($sql, [
            'customerId' => $customerId,
            'limit' => $limit,
            'offset' => $offset,
        ]);

        $total = $this->connection->fetchOne(
            'SELECT COUNT(*) FROM orders WHERE customer_id = :customerId',
            ['customerId' => $customerId]
        );

        return new PaginatedResult(
            items: array_map(fn (array $row) => OrderSummaryDTO::fromRow($row), $rows),
            total: (int) $total,
            page: $page,
            limit: $limit,
        );
    }
}
```

## Dispatching Queries

Driving adapters dispatch through the `QueryBusInterface` driving port, which
returns the handler result directly.

```php
use App\Shared\Application\Port\In\QueryBusInterface;

// In a driving adapter:
$userDTO = $this->queryBus->dispatch(new GetUserByIdQuery($id));
```

## Query Bus Helper Trait

This is a driving-adapter helper, so it lives with the other HTTP helpers in the
Shared driving-adapter namespace.

```php
namespace App\Shared\Infrastructure\Adapter\In\Http;

use Symfony\Component\Messenger\MessageBusInterface;
use Symfony\Component\Messenger\Stamp\HandledStamp;

trait QueryBusTrait
{
    private readonly MessageBusInterface $queryBus;

    protected function query(object $query): mixed
    {
        $envelope = $this->queryBus->dispatch($query);
        $handledStamp = $envelope->last(HandledStamp::class);
        return $handledStamp?->getResult();
    }
}
```

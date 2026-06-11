---
description: "Symfony Messenger async processing — message queues, retry strategies, failure transport, Symfony Scheduler, idempotency patterns, background jobs. Triggers on: messenger, async, queue, retry, scheduler, background job, worker, transport, message queue, cron, scheduled task"
---

# Symfony Messenger & Async Processing

You are an expert in async message processing with Symfony Messenger within hexagonal architecture.

## When to Activate

- User needs async processing or background jobs
- User asks about message queues or retry strategies
- User mentions cron jobs (redirect to Scheduler)
- User needs idempotency or failure handling
- User asks about Symfony Scheduler

## Core Principle: No Cron Jobs

**Never use raw cron.** Always use:
- **Symfony Messenger** for async message processing
- **Symfony Scheduler** for recurring tasks

## Async Command Pattern

```php
// The command is the same — async is a routing concern, not a code concern
namespace App\Report\Application\UseCase\Command\GenerateReport;

final readonly class GenerateReportCommand
{
    public function __construct(
        public string $reportId,
        public string $userId,
        public string $type,
    ) {
    }
}

// Handler is identical to sync — the bus handles async dispatch
final readonly class GenerateReportHandler implements CommandHandlerInterface
{
    public function __invoke(GenerateReportCommand $command): void
    {
        // Long-running operation
    }
}
```

```yaml
# Route to async transport
framework:
    messenger:
        routing:
            'App\Report\Application\UseCase\Command\GenerateReport\GenerateReportCommand': async
```

## Idempotency

Every async handler MUST be idempotent — safe to retry:

```php
final readonly class ProcessPaymentHandler implements CommandHandlerInterface
{
    public function __construct(
        private PaymentRepositoryInterface $paymentRepository,
        private PaymentGatewayInterface $paymentGateway,
    ) {
    }

    public function __invoke(ProcessPaymentCommand $command): void
    {
        // Check if already processed (idempotency)
        $existing = $this->paymentRepository->findByIdempotencyKey($command->idempotencyKey);
        if ($existing !== null) {
            return; // Already processed, skip
        }

        // Process payment
        $result = $this->paymentGateway->charge($command->customerId, $command->amount);
        $this->paymentRepository->save($result);
    }
}
```

## References

See `references/` for detailed guides:
- `messenger-config.md` — Full Messenger YAML configuration
- `idempotency-patterns.md` — Idempotency key strategies
- `scheduler-patterns.md` — Symfony Scheduler setup and patterns

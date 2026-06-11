---
description: "Symfony domain modeling — entities, value objects, domain events, aggregates, domain exceptions. Triggers on: entity, value object, domain event, aggregate, domain exception, domain model, domain logic, business rule, invariant"
---

# Symfony Domain Modeling

You are an expert in Domain-Driven Design within Symfony hexagonal architecture. Use this skill when users need to create or modify domain model elements.

## When to Activate

- User wants to create an entity or aggregate root
- User asks about value objects
- User needs domain events
- User mentions domain exceptions or business rules
- User discusses aggregates or invariants

## Entity Patterns

### Rules
- Private constructor + static factory method(s)
- No Doctrine annotations/attributes — mapping is in Infrastructure
- Record domain events on state changes
- Protect invariants in methods, not in external validators
- Use value objects for typed properties (no primitive obsession)

### Template
```php
namespace App\{Module}\Domain\Entity;

final class {Entity}
{
    private array $domainEvents = [];

    private function __construct(
        private {EntityId} $id,
        // typed properties using value objects
    ) {
    }

    public static function create(/* params */): self
    {
        // validate invariants
        $entity = new self(/* params */);
        $entity->recordEvent(new {Entity}Created(/* event data */));
        return $entity;
    }

    // Business methods that protect invariants
    public function doAction(/* params */): void
    {
        // guard clause / invariant check
        // state change
        $this->recordEvent(new {ActionDone}(/* event data */));
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

## Value Object Patterns

### Rules
- Always `final readonly class`
- Self-validating in constructor (throw domain exception if invalid)
- Immutable — no setters
- Implement `equals()` for comparison
- Use for: IDs, email, money, dates, status enums

### Template
```php
namespace App\{Module}\Domain\ValueObject;

use App\{Module}\Domain\Exception\Invalid{ValueObject}Exception;

final readonly class {ValueObject}
{
    public function __construct(
        public string $value,
    ) {
        if (/* validation fails */) {
            throw new Invalid{ValueObject}Exception($value);
        }
    }

    public function equals(self $other): bool
    {
        return $this->value === $other->value;
    }

    public function __toString(): string
    {
        return $this->value;
    }
}
```

## Domain Event Patterns

### Rules
- Past-tense naming: `UserRegistered`, `OrderPlaced`, `PaymentFailed`
- Immutable (`final readonly class`)
- Carry only the data needed by handlers (IDs, not full entities)
- Created inside entities on state changes

### Template
```php
namespace App\{Module}\Domain\Event;

final readonly class {PastTenseEvent}
{
    public function __construct(
        public string $entityId,
        public \DateTimeImmutable $occurredAt = new \DateTimeImmutable(),
        // additional event-specific data
    ) {
    }
}
```

## Domain Exception Patterns

```php
namespace App\{Module}\Domain\Exception;

final class {DomainException} extends \DomainException
{
    public static function because{Reason}(/* context */): self
    {
        return new self(sprintf('Descriptive message: %s', $context));
    }
}
```

## Single Home for an Invariant

A business invariant (e.g. `page >= 1`) lives in exactly one place: the Domain, inside a Value Object. A driving adapter must NOT re-implement the rule (e.g. `max(1, ...)`): every caller would duplicate it and could forget it. The VO is the sole guardian — Infrastructure sends raw values, the Domain decides.

```php
// Good: the rule lives once, in the VO
final readonly class PageNumber
{
    private function __construct(public int $value)
    {
    }

    public static function fromInt(int $value): self
    {
        if ($value < 1) {
            throw InvalidPageNumber::becauseItMustBeAtLeastOne($value);
        }

        return new self($value);
    }
}

// Bad: the adapter In recopies the rule
$page = max(1, (int) $request->query->get('page')); // rule duplicated, can drift
```

## References

See `references/` for detailed patterns:
- `entity-patterns.md` — Full entity examples with aggregates
- `value-object-patterns.md` — Common value object implementations
- `domain-event-patterns.md` — Event design and dispatch patterns

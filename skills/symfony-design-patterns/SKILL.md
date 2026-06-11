---
description: "Symfony design pattern selection — when and how to apply Factory, Specification, Strategy, State, Null Object, Decorator, Template Method, Adapter in DDD/hexagonal code. A decision guide that maps a problem shape to the right pattern, so patterns are applied only where they earn their place. Triggers on: design pattern, factory, specification, strategy, state machine, null object, decorator, template method, which pattern, pattern for, refactor conditionals, polymorphism, swappable algorithm, lifecycle, state transition"
---

# Symfony Design Pattern Selection

You are an expert in applying GoF and DDD patterns inside Symfony hexagonal architecture. Use this skill to decide WHICH pattern fits a problem and HOW to implement it cleanly. Apply patterns only where the problem shape calls for them — never force a pattern. Over-engineering violates SOLID just as much as under-design.

## Decision Table — problem shape to pattern

| Problem shape | Pattern | Layer |
|---------------|---------|-------|
| Object creation is complex, must protect invariants, or has multiple construction paths | **Factory** | Domain (static factory method) / Domain service |
| A reusable business rule that selects or validates entities, composable with AND/OR/NOT | **Specification** | Domain |
| Behavior must be swapped at runtime (pricing, shipping, payment provider) | **Strategy** | Domain or Application |
| An entity moves through a lifecycle with rules per state (Order: draft -> placed -> shipped) | **State** | Domain |
| You repeatedly null-check a collaborator and want a safe default | **Null Object** | Domain |
| Add behavior (logging, caching, metrics) around an existing port without changing it | **Decorator** | Infrastructure (adapter wrapping) |
| Several use cases share a skeleton with varying steps | **Template Method** | Application |
| Translate an external contract into a domain contract | **Adapter** | Infrastructure/Adapter/Out |
| Encapsulate a request as an object, dispatched to one handler | **Command** | Application (CQRS) |
| Something happened; zero-to-many parts must react, decoupled | **Observer** (domain events) | Domain emits / Application handles |
| A request passes through ordered steps, each may handle or pass | **Chain of Responsibility** | Application (pipeline/middleware) |
| Construction has many optional parts or a fluent assembly | **Builder** | Domain / test fixtures |
| Simplify a complex subsystem behind one entry point | **Facade** (use sparingly) | Infrastructure/Application |
| Control access to an object (lazy load, guard) | **Proxy** (Decorator usually fits better) | Infrastructure |
| Decouple collaborators that would otherwise call each other | **Mediator** (the message bus already is one) | Application |

## When NOT to apply a pattern

- One construction path, no invariants beyond the constructor -> plain `new`, no Factory.
- A single conditional that will not grow -> keep the `if`, no Strategy/State.
- Only one implementation of an interface ever expected -> no Strategy.
- Null is genuinely meaningful (absence is a domain concept) -> model it explicitly, not Null Object.

Forcing patterns adds indirection without payoff. Prefer the simplest design that protects the invariants.

## Factory

Use when creation is complex or must guarantee a valid object. Aligns with GRASP Creator and the project rule "entity = private constructor + named factory method".

```php
final class Order
{
    private function __construct(
        private readonly OrderId $id,
        private OrderStatus $status,
        private Money $total,
    ) {}

    public static function place(OrderId $id, Money $total): self
    {
        if ($total->isNegative()) {
            throw OrderCannotBePlaced::becauseTotalIsNegative();
        }

        return new self($id, OrderStatus::Placed, $total);
    }
}
```

Standalone factory (Domain service) when creation needs collaborators or external data. Never put persistence in a factory.

## Specification

Use to encapsulate a business rule as a first-class, composable object. Prefer over scattering the same `if` across handlers.

```php
interface Specification
{
    public function isSatisfiedBy(object $candidate): bool;
}

final class CustomerIsEligibleForDiscount implements Specification
{
    public function isSatisfiedBy(object $candidate): bool
    {
        return $candidate instanceof Customer
            && $candidate->totalSpent()->isGreaterThan(Money::fromInt(10000));
    }
}
```

Compose with And/Or/Not specifications. For persistence, translate the specification to a Criteria in the Doctrine adapter — never run raw SQL.

## Strategy

Use when an algorithm varies and must be selectable at runtime. Replaces `switch`/`instanceof` chains (OCP).

```php
interface ShippingCostStrategy
{
    public function cost(Order $order): Money;
}
```

Bind concrete strategies via `services.yaml`; inject the right one through a port. Adding a new strategy = new class, zero edits to existing code.

## State

Use for entity lifecycles with per-state rules. Prefer a `OrderStatus` enum with a `canTransitionTo()` method for simple cases; promote to State objects only when each state carries distinct behavior.

```php
enum OrderStatus
{
    case Draft;
    case Placed;
    case Shipped;

    public function canTransitionTo(self $next): bool
    {
        return match ($this) {
            self::Draft => $next === self::Placed,
            self::Placed => $next === self::Shipped,
            self::Shipped => false,
        };
    }
}
```

## Null Object

Use to remove repetitive null checks with a safe, do-nothing default. Implements the same port as the real collaborator.

```php
final class NullNotifier implements Notifier
{
    public function notify(UserId $user, Message $message): void
    {
    }
}
```

## Decorator

Use to add cross-cutting behavior around a port without touching the real adapter. Lives in Infrastructure, wraps the adapter, bound via `services.yaml` decoration.

```php
final class LoggingPaymentGateway implements PaymentGateway
{
    public function __construct(
        private readonly PaymentGateway $inner,
        private readonly LoggerInterface $logger,
    ) {}

    public function charge(Money $amount): PaymentResult
    {
        $this->logger->info('charging', ['amount' => $amount->amount()]);

        return $this->inner->charge($amount);
    }
}
```

## Template Method

Use when use cases share a fixed skeleton with a few varying steps. Keep the skeleton in Application; do not leak it into Domain.

## Command

Encapsulate an intent as an immutable object handled by exactly one handler. In this project, CQRS commands ARE this pattern: `final readonly` command in `Application/UseCase/Command/{Name}/`, colocated `__invoke` handler, dispatched via the command bus. Return void or an id.

## Observer

Decouple "something happened" from the reactions. Use domain events, not a manual subject/observer wiring: the entity records a past-tense event; Application event handlers react. Side-effects (email, cache, notifications) happen ONLY in event handlers, never in command handlers.

```php
final class Order
{
    public function place(): void
    {
        // ... state change ...
        $this->recordThat(new OrderPlaced($this->id));
    }
}
```

## Chain of Responsibility

A request flows through ordered links; each may handle it or pass it on. Fits validation pipelines and middleware. Prefer Symfony's built-in middleware/messenger middleware over a hand-rolled chain when possible.

## Builder

Use for assembling an object with many optional parts, or for readable test fixtures. In Domain, prefer a named factory method for the common case and reserve Builder for genuinely complex assembly. Builders are especially useful as test data builders.

## Facade / Proxy / Mediator (use sparingly)

- **Facade**: a single simple entry point over a complex subsystem. Often an Application service already plays this role — do not add a Facade just to wrap one call.
- **Proxy**: controls access (lazy loading, guarding). For cross-cutting behavior (logging, caching) prefer **Decorator**; reach for Proxy only when you must control *access* rather than *augment* behavior.
- **Mediator**: decouples collaborators through a hub. The command/query bus is already your Mediator — do not build another.

## Patterns deliberately NOT promoted

Singleton (the DI container manages lifecycles — a Singleton is almost always a smell here), Prototype, Abstract Factory, Bridge, Composite, and Iterator (PHP generators / native `Iterator` cover it). These have rare, niche uses in app-level DDD. Do not introduce them unless a concrete problem clearly demands it — adding them is usually accidental complexity.

## Pattern selection checklist

When building or refactoring a feature, ask in order:
1. Does creation need protection or branching? -> Factory.
2. Is there a reusable rule I am about to duplicate? -> Specification.
3. Is there a `switch`/`instanceof` on type that will grow? -> Strategy or polymorphism.
4. Does an entity have states with different rules? -> State (enum-first).
5. Am I null-checking the same collaborator repeatedly? -> Null Object.
6. Do I need to wrap a port with logging/caching/metrics? -> Decorator.
7. None of the above? -> Do not add a pattern.

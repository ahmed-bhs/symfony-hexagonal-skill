# Symfony Hexagonal Architecture Rules

This plugin enforces hexagonal architecture in Symfony projects. These rules apply to ALL code generation and review.

## Progressive Refactoring Strategy

**This plugin does NOT require rewriting an entire project at once.** When added to an existing Symfony project that doesn't follow hexagonal architecture:

1. **Detect existing structure first**: Before any code generation, scan the project to understand current conventions (traditional Symfony, MVC, etc.)
2. **New code follows hexagonal rules**: All newly created modules/features use the hexagonal structure
3. **Ask before refactoring existing code**: When the user requests a change that touches non-hexagonal code, ASK:
   - "Bu modül henüz hexagonal yapıda değil. İsterseniz önce `{Module}` modülünü hexagonal mimariye taşıyıp sonra yeni özelliği ekleyeyim. Ya da mevcut yapıda devam edebilirim. Hangisini tercih edersiniz?"
   - Offer clear options: (a) refactor to hexagonal first, then add feature, (b) add feature in current structure, (c) add feature in hexagonal but leave existing code as-is
4. **Refactor only the touched module**: Never refactor unrelated modules. Only the module the user is actively working on is a candidate.
5. **Incremental migration path**: When user chooses refactoring, migrate one layer at a time — Domain first, then Application, then driven adapters (Adapter/Out), finally driving adapters (Adapter/In).

### When to Ask
- User wants to modify an existing controller/service/repository that isn't in hexagonal structure
- User wants to add a feature to a module that has mixed architecture
- User asks to create something that interacts with legacy (non-hexagonal) code

### When NOT to Ask
- User explicitly says "just do it" or "mevcut yapıda devam et"
- Project is already fully hexagonal
- User is creating a brand-new module with no legacy dependencies
- Trivial changes (config, env, docs) that don't involve architecture

## 4 Core Rules (NEVER violate)

### 1. Dependency Rule
- `Domain/` has ZERO framework *logic* — no Symfony/Doctrine services, no third-party behaviour. Inert mapping/validation attributes are the documented exception (see below)
- Domain contains plain-PHP business code: entities, value objects, events, exceptions, port interfaces. Inert mapping/validation attributes (`#[ORM\...]`, format `#[Assert\...]`) are allowed; framework *services/logic* are not
- Only `Application/` may depend on `Domain/`. `Infrastructure/Adapter/Out` implements ports; `Infrastructure/Adapter/In` (driving) calls application services via the bus.
- Direction: Adapter/In → Application → Domain ← Adapter/Out (no separate Presentation layer)

### 2. Port/Adapter Enforcement
- Every external interaction (DB, API, filesystem, email) MUST go through a port interface in `Domain/{Module}/Port/`
- Adapters implementing ports live in `Infrastructure/{Module}/`
- Bind adapters to ports via `services.yaml` — never inject concrete adapters directly

### 3. CQRS Separation
- Commands: `final readonly class` in `Application/UseCase/Command/{Name}/`, return void or ID
- Queries: `final readonly class` in `Application/UseCase/Query/{Name}/`, return DTO
- Handlers: one `__invoke` method, colocated with their message. Bind via marker interface (`CommandHandlerInterface`/`QueryHandlerInterface`) + bus port; `#[AsMessageHandler]` is an accepted alternative
- Two buses: `command.bus` and `query.bus` in messenger config; bus interfaces are driving ports in `Shared/Application/Port/In/`
- Port Out placement depends on who expresses the need: a domain need → `Domain/Port/Out/`; a use-case need → `Application/Port/Out/`. Read-only ports use the `QueryInterface` suffix; mutating aggregate ports use `RepositoryInterface`
- A handler NEVER passes the whole Query/Command to a port. It unwraps it: pass a Domain VO when the params carry a business rule or derived calculation (`page >= 1`, `offset`), otherwise pass primitives. Passing an Application Query/Command object to a port Out makes the Domain import the Application — a forbidden arrow (the `symfony-leaky-abstractions` skill flags it; enforce with deptrac when configured)
- Write side always goes through the aggregate: the Command handler builds the aggregate, applies invariants, then `save(Aggregate)`. The bare-props/VO choice applies only to reads
- A business invariant lives in exactly one place — the Domain VO. Driving adapters send raw values and must not recopy the rule (no `max(1, ...)` in a controller)

### 4. Event-Driven Side-Effects
- Domain events: past-tense named, immutable, created within entities
- Event handlers in `Application/{Module}/EventHandler/`
- Side-effects (email, notifications, cache) triggered ONLY via events — never in command handlers

## Additional Standards

### Engineering Principles (apply to ALL code, always)
- **SOLID**: SRP (one reason to change — no God classes, no `Manager`/`AndDoX` names); OCP (extend via new classes/strategies, never `switch (instanceof)`); LSP (implementations interchangeable, no side-effects); ISP (thin focused ports, no god interface); DIP (Domain/Application depend on abstractions only, never concretions).
- **GRASP**: Information Expert (behavior lives with its data), Creator (factories create objects), Low Coupling / High Cohesion, Protected Variations (hide change behind ports).
- **No primitive obsession**: every domain concept is a Value Object (`Email`, `Money`, `OrderId`) — never bare `string`/`int`/`array`. VOs are `final readonly`, self-validating in the constructor, with `equals()`.
- **Simplicity**: KISS (simplest design that protects the invariants), YAGNI (build only what the feature needs now), avoid accidental complexity, reduce cognitive load, prefer readability over cleverness.
- **Design discipline**: Separation of Concerns; composition over inheritance; program to an interface, not an implementation (interface-driven design); make dependencies explicit (constructor injection, no service locator); explicit module boundaries.
- **Object design**: Tell, Don't Ask (rich behavior on entities, not anemic getters + external logic); Law of Demeter (no train-wreck `->a()->b()->c()`); information hiding / encapsulation (no public setters that break invariants); prefer immutability for value objects; avoid temporal coupling (no required call order).
- **Robustness**: fail fast (reject invalid input at the boundary); make invalid states unrepresentable (model with enums/VOs/types so an illegal value cannot be constructed); principle of least astonishment (clear naming, predictable behavior).
- **Clarity**: keep business rules close to the domain; document decisions (the why), not obvious code.
- **Patterns by need**: apply patterns only where the problem shape calls for them (see the `symfony-design-patterns` skill). Forcing a pattern is over-engineering and violates KISS/YAGNI/SOLID.

### API Response Format
All API endpoints return: `{"result": mixed, "error": ?object, "extra": ?object, "status": int}`
Debug details included only when `APP_DEBUG=true`.

### No Cron Jobs
Use Symfony Messenger async transport + Symfony Scheduler component. Never raw cron.

### Security
Every endpoint MUST have a ROLE via `#[IsGranted]` or Voter. Use role hierarchy in `security.yaml`. Complex authorization → Symfony Voter.

### Quality
- PHPStan level 8+ mandatory
- DTOs at every boundary (controller input, handler output, API response)
- Validation at 3 layers: driving adapter (input), Application (DTO), Domain (invariants)
- Quality gates are blocking before integration: coding style, static analysis, migrations, and tests must all pass (enforced locally via GrumPHP or similar and in CI). Layer dependencies should be enforced by deptrac when configured (forbid `Domain → Application` and `Application → Infrastructure`)
- Business rules live in the Domain or Application — never in technical validators or the UI

### UI / Admin Adapters
- API Platform writes keep the HTTP contract separate from the application use case (the resource/input DTO is not the Command)
- EasyAdmin is an accepted pragmatic compromise, but CRUD controllers hold NO business logic — they dispatch commands/queries like any other driving adapter

### Async Resilience
- Messages must be idempotent (use idempotency keys)
- Configure retry with backoff in `messenger.yaml`
- Failed messages → `failed` transport with monitoring

### Doctrine
- Repository adapters in `Infrastructure/Adapter/Out/Doctrine/`
- Entity mapping via `#[ORM\...]` attributes on the Domain entity (inert metadata, allowed) or externalised XML in Infrastructure. DB-access validators (`UniqueEntity`, `Callback`, `Expression`) must NOT be in Domain — put them in `Adapter/In/Validation/`
- Ask user preference for migration strategy (manual vs auto-diff)
- **NEVER use native/raw SQL** — no `$connection->executeQuery()`, `$connection->executeStatement()`, `NativeQuery`, `$connection->prepare()`, or raw SQL strings in application code
- Always use Doctrine QueryBuilder (ORM or DBAL), DQL, Repository finder methods, or Criteria API
- **Only exception**: Doctrine Migrations (`$this->addSql()`) — the migration API requires raw SQL by design
- When reviewing or generating code, flag any native SQL usage as a CRITICAL violation

## Directory Structure (per module)

```
src/{Module}/
├── Domain/Entity/, ValueObject/, Event/, Exception/, Port/Out/
├── Application/UseCase/Command/{Name}/, UseCase/Query/{Name}/, Port/Out/, DTO/, EventHandler/
└── Infrastructure/Adapter/
    ├── In/ApiPlatform/, CLI/, Admin/, Security/, Validation/   # driving
    └── Out/Doctrine/, Storage/, Bus/                           # driven
```

Module-first namespace: `App\{Module}\{Layer}\...`. The command/query/event buses
are driving ports in `Shared/Application/Port/In/`.

## Code Generation Checklist
When generating code, verify:
- [ ] No Symfony/Doctrine imports in Domain/
- [ ] External calls go through port interfaces
- [ ] Commands and Queries are separate with dedicated handlers
- [ ] Side-effects use domain events, not direct calls
- [ ] API responses use standard payload format
- [ ] Endpoints have ROLE authorization
- [ ] DTOs used at boundaries
- [ ] Tests follow src/ mirror structure
- [ ] No native/raw SQL in application code (only QueryBuilder, DQL, Criteria)

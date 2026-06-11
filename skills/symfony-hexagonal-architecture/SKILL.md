---
description: "Symfony hexagonal architecture setup — project structure, module scaffolding, layer responsibilities, dependency rules. Triggers on: architecture, module, layer, hexagonal, scaffold, project structure, directory structure, new project, new module"
---

# Symfony Hexagonal Architecture

You are an expert in Symfony hexagonal architecture (ports & adapters). Use this skill when users need to set up project structure, create new modules, or understand layer responsibilities.

## When to Activate

- User wants to create a new Symfony project with hexagonal architecture
- User asks about project structure or directory layout
- User wants to scaffold a new module/bounded context
- User asks about layer responsibilities or dependency rules
- User mentions "hexagonal", "ports and adapters", "clean architecture" in Symfony context

## Core Principles

### Layer Direction (STRICT)
```
Adapter/In → Application → Domain ← Adapter/Out
```

Driving adapters (controllers, CLI, API Platform, EasyAdmin) call Application.
Application orchestrates Domain. Driven adapters (Doctrine, storage, buses,
external clients) implement ports. Domain NEVER depends on Infrastructure.

There is **no separate `Presentation/` layer**. Entry points are driving
adapters — same nature as a Doctrine repository, only the flow direction
differs. Everything framework-facing lives under `Infrastructure/Adapter/`.

### Module Structure

Module-first: every bounded context owns its three layers under `src/{Module}/`.

```
{Module}/Domain/
├── Entity/          # Aggregates and entities (pure PHP)
├── ValueObject/     # Immutable value objects
├── Event/           # Domain events (past-tense)
├── Exception/       # Domain-specific exceptions
└── Port/Out/        # Driven ports the domain logic needs (repositories…)

{Module}/Application/
├── UseCase/Command/{UseCaseName}/   # {UseCaseName}Command + Handler colocated
├── UseCase/Query/{UseCaseName}/     # {UseCaseName}Query + Handler colocated
├── Port/Out/        # Technical driven ports (storage, token gen, read models)
├── DTO/             # Data transfer objects
└── EventHandler/    # Domain event handlers

{Module}/Infrastructure/Adapter/
├── In/              # Driving adapters — channels + shared driving concerns
│   ├── ApiPlatform/ # REST channel: resources, processors, providers
│   │   └── Validation/  # constraints SPECIFIC to the API channel
│   ├── CLI/         # Console channel
│   ├── Admin/       # EasyAdmin channel: CRUD, LiveComponents
│   │   └── Validation/  # constraints SPECIFIC to the Admin channel
│   ├── Validation/  # constraints SHARED across channels (API + Admin)
│   └── Security/    # Voters, user checkers, auth handlers (shared driving concern)
└── Out/             # Driven adapters
    ├── Doctrine/    # Repositories + Mapping/ (XML/PHP, never in Domain)
    ├── Storage/     # Filesystem, S3, mailer
    └── Bus/         # Messenger bus implementations
```

The command/query/event **buses are the driving ports** of the system. They
live in a shared module at `Shared/Application/Port/In/`; their Messenger
adapters live at `Shared/Infrastructure/Adapter/Out/Bus/`.

**`Adapter/In/` holds inbound channels plus shared driving concerns — never duplicate.**
The channel subfolders are the ways the world drives the app: `ApiPlatform/` (HTTP),
`CLI/`, `Admin/`. Validation and Security are driving concerns, not channels; place
each one by its **scope of reuse**:
- **Shared across channels** (same voter, or the same constraint used by both the API
  and the Admin) → hoist to a cross-channel folder: `In/Security/`, `In/Validation/`.
  Don't duplicate a shared rule into each channel.
- **Channel-specific** (an API-only payload constraint, an Admin-only form constraint)
  → nest inside the channel that triggers it: `ApiPlatform/Validation/`, `Admin/Validation/`.

The decision is the same one as for shared kernels: hoist what is shared, localise what
is specific. Either way these stay under `Adapter/In/` (they are driving concerns) —
not in a `Framework/` or `Out/` folder.

## Progressive Refactoring (Existing Projects)

When working on an existing project that doesn't follow hexagonal architecture:

### Step 1: Detect Current Structure
Before writing any code, check:
```
- Is src/ organized by layer (Domain/, Application/, etc.) or by traditional Symfony (Controller/, Entity/, Repository/)?
- Does the module the user is working on already follow hexagonal patterns?
- Are there Doctrine annotations directly on entities?
```

### Step 2: Ask the User
When the user requests a feature/change in a non-hexagonal module, ALWAYS ask:

> "Bu modül (`{Module}`) şu anda hexagonal yapıda değil. Şu seçeneklerden birini tercih edebilirsiniz:
> 1. **Önce refactor**: `{Module}` modülünü hexagonal mimariye taşıyıp, sonra yeni özelliği ekleyeyim
> 2. **Sadece yeni kodu hexagonal yap**: Mevcut koda dokunmadan, yeni eklenen kısımları hexagonal yapıda oluşturayım
> 3. **Mevcut yapıda devam et**: Bu sefer hexagonal yapıyı atlayalım
>
> Hangisini tercih edersiniz?"

### Step 3: Incremental Migration (if user chooses refactor)
Migrate one layer at a time, in this order:

1. **Domain first**: Extract entities to `{Module}/Domain/Entity/`, create value objects, define port interfaces in `Domain/Port/Out/`
2. **Application second**: Create use-case folders `Application/UseCase/Command/{Name}/` and `Query/{Name}/`, move business logic out of controllers/services
3. **Driven adapters third**: Move Doctrine repositories to `Infrastructure/Adapter/Out/Doctrine/`, extract mappings
4. **Driving adapters last**: Thin out controllers into `Infrastructure/Adapter/In/`, add `ApiResponseTrait`, apply `#[IsGranted]`

After each layer, verify the code still works before moving to the next.

### Key Principles
- **Never touch modules the user isn't working on** — only refactor what's in scope
- **One module at a time** — don't try to migrate the whole project
- **Keep the app working** — each step should be a working state
- **Respect user's pace** — some teams prefer gradual migration over months

## Scaffolding a New Module

When user asks to create a new module, generate ALL directories and base files:

1. Create directory structure (Domain, Application, Infrastructure/Adapter/{In,Out})
2. Create a sample port interface in `{Module}/Domain/Port/Out/`
3. Create a sample repository adapter in `{Module}/Infrastructure/Adapter/Out/Doctrine/`
4. Bind the port to adapter in `services.yaml`
5. Suggest initial entities, and use-case folders (`Application/UseCase/Command|Query/{Name}/`) based on the module purpose

## Screaming Architecture

The tree must **scream the business, not the framework**. A newcomer opening
`src/` should read the bounded contexts and their capabilities first — not
"Controllers", "Entities", "Services".

- **Top level = business modules**, never technical layers. `src/User/`,
  `src/DepositRequest/`, `src/EcoOrganization/` — not `src/Controller/`,
  `src/Repository/`. This is why the namespace is module-first.
- **Use-case folders name capabilities.** `Application/UseCase/Command/ApproveDepositRequest/`
  tells you what the system *does*. The colocated `Command` + `Handler` are the
  vertical slice of that one capability — one folder, one diff, one reason to
  change. A flat `Command/` bag of 50 files screams nothing.
- **Adapters name the channel, not the plumbing.** `Adapter/In/ApiPlatform/`,
  `Adapter/In/Admin/` say "this is how the world drives us"; `Adapter/Out/Doctrine/`,
  `Adapter/Out/Storage/` say "this is how we reach the world".

When choosing a directory name, prefer the word that describes the *intent*
(the use case, the context) over the word that describes the *mechanism*.

## Key Rules

1. **Domain purity**: No framework *logic* in Domain — no calls into Symfony/Doctrine services, no infrastructure behaviour. **Declarative mapping/validation attributes are allowed**: `#[ORM\...]` and format `#[Assert\...]` are inert metadata that do not change behaviour and keep the entity testable as plain PHP. Forbidden in Domain: DB-access constraints (`#[UniqueEntity]`, `#[Assert\Callback]`, `#[Assert\Expression]`) — those imply a query/service call and belong in `Adapter/In/Validation/`. Whitelist the mapping vendors in deptrac (`MappingVendors`).
2. **Port-first**: Define the interface before the implementation
3. **One module = one bounded context**: Modules communicate via events, not direct calls
4. **Namespace convention**: `App\{Module}\Domain\...`, `App\{Module}\Application\...`, etc. (module-first)
5. **No Presentation layer**: entry points are driving adapters under `Infrastructure/Adapter/In/`

## References

See `references/` for detailed guides:
- `directory-structure.md` — Full directory layout with file examples
- `layer-responsibilities.md` — What belongs in each layer
- `dependency-rules.md` — Import rules and PHPStan enforcement

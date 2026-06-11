---
description: "Symfony leaky abstraction detection — finds framework/infrastructure details leaking through hexagonal boundaries: Doctrine/Symfony types in Domain, ORM attributes or Collection in port interfaces, HTTP Request/Response in Application, untranslated infrastructure exceptions. Triggers on: leaky abstraction, framework leakage, Doctrine in domain, Symfony in domain, port leak, interface leak, Collection in interface, Request in handler, exception translation, layer violation, boundary violation, domain purity"
---

# Symfony Leaky Abstractions

You detect leaky abstractions in Symfony hexagonal projects (module-first: `src/{Module}/{Domain,Application,Infrastructure}`). Use this skill when reviewing code, checking domain purity, or when the user mentions layer/boundary violations or framework leaking into the core. Report each leak with severity and the clean replacement.

Note: this project's CLAUDE.md ALLOWS inert mapping/validation attributes (`#[ORM\...]`, format `#[Assert\...]`) on Domain entities. Do NOT flag those. DO flag framework *services/logic*, DB-access constraints (`UniqueEntity`, `Callback`, `Expression`), and infrastructure types crossing boundaries.

## Detection Patterns

### Framework services/logic in Domain (CRITICAL)
```bash
# Framework services (not inert attributes)
grep -rn "EntityManager\|Connection\|QueryBuilder\|HttpClient\|Mailer\|HttpFoundation" --include="*.php" src/*/Domain/
grep -rn "use Symfony\\\\Component\\\\\|use Illuminate\\\\" --include="*.php" src/*/Domain/
# DB-access constraints leaking into Domain (allowed attributes are inert; these are not)
grep -rn "UniqueEntity\|Assert\\\\Callback\|Assert\\\\Expression" --include="*.php" src/*/Domain/
# Request/Response in Domain
grep -rn "Request\|Response" --include="*.php" src/*/Domain/
```

### Port interface leakage (CRITICAL)
Ports live in `Domain/{Module}/Port/`. They must expose domain types only.
```bash
# Doctrine collection / query types in ports
grep -rn "Collection\|ArrayCollection\|QueryBuilder\|EntityManager\|Connection\|PDO" --include="*Interface.php" src/*/Domain/
grep -rn "Doctrine\\\\\|Symfony\\\\\|Redis\|Guzzle\|Http" --include="*Interface.php" src/*/Domain/

# Domain importing the Application layer — a forbidden arrow.
# Most common cause: a port Out typed with an Application Query/Command instead of
# unwrapped primitives or a Domain VO (see symfony-cqrs-handlers "Passing Query parameters").
grep -rn "use App\\\\.*\\\\Application\\\\" --include="*.php" src/*/Domain/
```

### HTTP in Application (CRITICAL)
```bash
# Request/Response in handlers/use cases — should take Command/Query DTO, return DTO/id
grep -rn "Request \$\|: Response\|: JsonResponse" --include="*.php" src/*/Application/
```

### Concrete return types from abstractions (WARNING)
```bash
# Interface returning a concrete class instead of a port/domain type
grep -rn "public function.*): [A-Z][a-z]*[A-Z]" --include="*Interface.php" src/
# Collection instead of array
grep -rn "): Collection\|): ArrayCollection" --include="*Interface.php" src/*/Domain/
```

### Exception leakage (WARNING)
```bash
# Infra exceptions thrown/caught in Domain or Application
grep -rn "PDOException\|Doctrine\\\\" --include="*.php" src/*/Domain/ src/*/Application/
# Adapter not translating to domain exception
grep -rn "flush()" --include="*.php" src/*/Infrastructure/Adapter/Out/
```

### Serialization leakage (WARNING)
```bash
grep -rn "JsonSerializable\|#\\[Groups\|#\\[ApiResource\|#\\[ApiProperty" --include="*.php" src/*/Domain/
```

## Clean Replacements

| Leaky | Clean |
|-------|-------|
| `Collection` in port | `array` (with `@return X[]` docblock) |
| `QueryBuilder` in port | `Criteria` or a Specification |
| `EntityManager` in port | the repository Port interface |
| `Request` in handler | a Command/Query DTO |
| `Response` in handler | return value + driving adapter formats it |
| `PDOException` crossing boundary | translate to a domain exception in the adapter |

## Exception Translation (adapter pattern)
```php
try {
    $this->em->persist($order);
    $this->em->flush();
} catch (UniqueConstraintViolationException $e) {
    throw OrderAlreadyExists::withId($order->id());
}
```

## Report Format
```markdown
# Leaky Abstractions Report
| Leak type | Critical | Warning |
|-----------|----------|---------|
| ... | n | n |

## [SEVERITY] {type}: {title}
- File: {path}:{line}
- Leak: {what}
- Fix: {clean replacement}
```

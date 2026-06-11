---
description: "Symfony bounded context analysis — detects cross-context coupling, direct foreign-entity references, missing anti-corruption layers, unintended shared kernel, and ubiquitous-language clashes in module-first DDD projects. Produces a context map and boundary fixes. Triggers on: bounded context, context map, cross-context, coupling between modules, shared kernel, anti-corruption layer, ACL, ubiquitous language, module boundary, foreign entity, context isolation, strategic DDD"
---

# Symfony Bounded Contexts

You analyze bounded context boundaries in Symfony module-first DDD projects (`src/{Module}/{Domain,Application,Infrastructure}` — each top-level module IS a bounded context). Use this skill when the user asks about module coupling, context boundaries, or strategic DDD. Report cross-context violations with severity and the correct integration pattern, and draw a Mermaid context map.

## Detection Patterns

### Discover contexts
```bash
# Each module = a bounded context
ls -d src/*/Domain/ 2>/dev/null
```

### Cross-context coupling (CRITICAL)
A module's Domain must not import another module's Domain. Cross-context links go via ID references or domain events.
```bash
for ctx in $(find src -maxdepth 1 -mindepth 1 -type d); do
  name=$(basename "$ctx")
  grep -rn "use App\\\\.*\\\\Domain\\\\" --include="*.php" "$ctx/Domain" 2>/dev/null | grep -v "use App\\\\$name\\\\"
done
```

### Foreign entity reference instead of ID (CRITICAL)
```bash
# Aggregates should hold UserId, not User (from another context)
grep -rn "private.*readonly [A-Z][a-z]*\s*\$" --include="*.php" src/*/Domain/Entity/
# Good: private readonly UserId $userId    Bad: private readonly User $user
```

### Missing anti-corruption layer (CRITICAL)
```bash
# External vendor models used directly in Domain — wrap in an adapter/translator
grep -rn "use Stripe\\\\\|use ExternalApi\\\\\|use Legacy\\\\" --include="*.php" src/*/Domain/
```

### Unintended shared kernel (WARNING)
```bash
# Same VO duplicated across contexts without an explicit Shared module
find src -path "*/Domain/ValueObject/*" -name "Email.php" -o -name "Money.php" -o -name "Address.php"
ls -d src/Shared 2>/dev/null
```

### Ubiquitous language clash (WARNING)
```bash
# Same class name, different meaning across contexts
for n in Account Order User Customer Client; do
  found=$(find src -path "*/Domain/*" -name "$n.php"); [ "$(echo "$found" | grep -c .)" -gt 1 ] && echo "AMBIGUOUS: $n in multiple contexts"
done
```

## Integration Patterns (the correct ways to cross a boundary)

| Pattern | When | How |
|---------|------|-----|
| ID reference | associate entities across contexts | hold `OtherId` VO, resolve via query |
| Domain event | async notify another context | publish past-tense event, other context's EventHandler reacts |
| Anti-corruption layer | external/legacy systems | adapter translates foreign model to domain model |
| Shared kernel | truly common VOs (Money, Currency) | explicit `src/Shared/`, VOs only, no entities |

## Anti-patterns

| Anti-pattern | Fix |
|--------------|-----|
| Direct foreign entity reference | replace with ID VO + query |
| Cross-context repository use | use events or an ACL |
| Synchronous cross-context call | async domain event |
| Shared entity across contexts | split, assign clear ownership |

## Report Format
```markdown
# Bounded Context Report

## Context Map
\`\`\`mermaid
graph TB
  subgraph Order
    O[Domain]
  end
  subgraph User
    U[Domain]
  end
  O -->|ID only| U
\`\`\`

| Context | Entities | Events | Depends on |
|---------|----------|--------|-----------|

## [SEVERITY] BC-xxx: {title}
- File: {path}:{line}
- Issue: {coupling}
- Fix: {integration pattern}
```

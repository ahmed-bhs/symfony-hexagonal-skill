---
description: "Symfony SOLID violation analysis — detects God classes (SRP), type switches/instanceof chains (OCP), broken contracts (LSP), fat interfaces (ISP), and concrete dependencies (DIP) in hexagonal PHP code. Reports violations with severity and remediation. Triggers on: SOLID, SRP, OCP, LSP, ISP, DIP, God class, single responsibility, open closed, type switch, fat interface, dependency inversion, code smell, refactor, too many dependencies, large class, violation"
---

# Symfony SOLID Analysis

You detect SOLID principle violations in Symfony hexagonal projects (module-first layout: `src/{Module}/{Domain,Application,Infrastructure}`). Use this skill when reviewing code, auditing quality, or when the user mentions SOLID, code smells, or refactoring. Report each finding with severity (CRITICAL / WARNING / INFO), file:line, and the remediation pattern.

## Detection Patterns

### SRP — Single Responsibility
```bash
# God classes (>500 lines = CRITICAL, 300-500 = WARNING)
find src -name "*.php" -exec wc -l {} \; | awk '$1 > 300 {print $0}'

# Too many public methods (>15)
for f in $(find src -name "*.php"); do c=$(grep -c "public function" "$f"); [ "$c" -gt 15 ] && echo "WARNING: $f ($c public methods)"; done

# Problematic names + "And" in class names
grep -rn "class .*\(Manager\|Helper\|Util\|And[A-Z]\)" --include="*.php" src/

# >7 constructor dependencies
grep -rn "__construct" --include="*.php" -A 20 src/ | grep -E "private|readonly"
```

### OCP — Open/Closed
```bash
# Type switches and instanceof chains (apply Strategy/polymorphism instead)
grep -rn "switch.*->type\|match.*::class\|match.*->type" --include="*.php" src/
grep -rn "if.*instanceof\|elseif.*instanceof" --include="*.php" src/
grep -rn "\[.*::class\s*=>" --include="*.php" src/
```

### LSP — Liskov Substitution
```bash
# Subtypes that break the contract
grep -rn "throw.*NotImplemented\|throw.*NotSupported\|throw.*UnsupportedOperation" --include="*.php" src/
# Empty overrides
grep -rn "public function.*{\s*}" --include="*.php" src/
```

### ISP — Interface Segregation
```bash
# Fat ports (>5 methods = WARNING, >8 = CRITICAL). Ports live in Domain/{Module}/Port/
for f in $(grep -rl "^interface" --include="*.php" src/); do c=$(grep -c "public function" "$f"); [ "$c" -gt 5 ] && echo "$f: $c methods"; done
# Generic interface names
grep -rn "interface \(Service\|Manager\|Handler\)\s*$" --include="*.php" src/
```

### DIP — Dependency Inversion
```bash
# Direct instantiation in Domain/Application (excluding VOs, exceptions, dates)
grep -rn "new [A-Z]" --include="*.php" src/*/Domain src/*/Application | grep -v "Exception\|DateTime\|DateTimeImmutable"
# Concrete type hints instead of ports
grep -rn "__construct" --include="*.php" -A 15 src/ | grep -E "(private|readonly) [A-Z]" | grep -v "Interface\|Abstract"
# Service locator anti-pattern
grep -rn "container->get\|\$this->get(" --include="*.php" src/
```

## Severity

| Severity | Criteria | Action |
|----------|----------|--------|
| CRITICAL | >500 LOC, >10 deps, NotImplementedException, type switch on growing types | Immediate refactor |
| WARNING | 300-500 LOC, 7-10 deps, fat interface | Plan refactor |
| INFO | 200-300 LOC, minor DIP | Monitor |

## Remediation Map

| Violation | Fix |
|-----------|-----|
| God class | Extract use cases / domain services |
| Type switch (OCP) | Strategy pattern (see `symfony-design-patterns`) |
| Fat interface | Segregate into Reader/Writer ports |
| Concrete dependency | Depend on a Port interface |
| Missing factory | Named static factory on the entity |

## When a Violation Is Acceptable (false positives)

- DTOs / Value Objects with many properties — single responsibility is data transfer.
- Doctrine entities mixing persistence-shaped fields with domain methods — framework convention.
- Thin driving adapters (controllers) that validate + delegate + respond.

## Report Format

```markdown
# SOLID Violations Report
| Principle | Critical | Warning | Info |
|-----------|----------|---------|------|
| SRP/OCP/LSP/ISP/DIP | n | n | n |

## [SEVERITY] {PRINCIPLE}: {title}
- File: {path}:{line}
- Issue: {what}
- Fix: {pattern}
```

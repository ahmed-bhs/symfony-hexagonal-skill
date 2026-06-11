---
description: "Audit the test pyramid of a branch — counts unit / integration / functional / Behat E2E tests, checks the pyramid shape (many unit, fewer integration, fewest E2E), verifies coverage of changed code, and flags test-double misuse (mocking internal domain instead of ports). Produces a CRITICAL/WARNING/INFO report with a verdict. Read-only."
allowed-tools: Read, Glob, Grep, Bash, Task
argument-hint: "[base-branch] [-- focus instructions]"
---

# Audit Tests (Test Pyramid)

Audit the test strategy of the current branch: the balance between test levels, coverage of the changed code, and correct use of test doubles. Read-only — never modify code. Lean on the `symfony-testing` skill for the project's testing conventions.

## What the test pyramid prescribes (the standard this audit enforces)

- **Shape**: many fast unit tests at the base, fewer integration tests in the middle, very few slow end-to-end tests at the top. Target guideline ≈ **unit 70% / integration 20% / E2E 10%** — treat as guidance, not a hard gate.
- **Ordering invariant** (hard): `unit > integration > (functional + Behat E2E)`. A violation means an **inverted pyramid** (top-heavy) or an **ice-cream cone** (E2E-heavy) — both CRITICAL anti-patterns: slow, brittle, expensive feedback.
- **Each level tests the right thing**: unit = one unit in isolation; integration = real adapters against a real backing service (same DB engine as prod); E2E = critical business journeys only, not edge cases (edge cases belong in unit tests).
- **Doubles**: mock/stub only at boundaries — the **ports (interfaces)**. NEVER mock internal domain collaborators (entities, value objects, domain services). Over-mocking couples tests to implementation.

## Step 1 — Scope

```bash
base="${1:-main}"
git rev-parse --verify "$base" >/dev/null 2>&1 || base="master"

# Changed production + test files on this branch
git diff --name-only "$base"...HEAD -- '*.php'
```

Audit the whole test suite for shape, and the changed production files for coverage.

## Step 2 — Classify and count tests

Use the project test layout (mirrors `src/`, module-first):
- `tests/Unit/.../Domain/` — Domain unit (pure PHP)
- `tests/Unit/.../Application/` — handler unit (ports mocked)
- `tests/Integration/` — real DB / real adapters
- `tests/Functional/` — full HTTP stack
- `tests/Behat/` (`.feature`) — E2E business scenarios

```bash
# Count test methods per level (a test = a test method, not a file)
unit_dom=$(grep -rl "extends TestCase" tests/Unit/*/Domain 2>/dev/null | xargs grep -hc "public function test\|#\[Test\]" 2>/dev/null | paste -sd+ | bc 2>/dev/null)
# Repeat the same counting for: tests/Unit/*/Application, tests/Integration, tests/Functional
# Behat scenarios:
behat=$(grep -rh "Scenario:" tests/Behat 2>/dev/null | wc -l)
```

Compute counts: `unit`, `integration`, `functional`, `behat`. Treat `e2e = functional + behat`.

## Step 3 — Evaluate the shape

1. **Ordering invariant** (CRITICAL if broken): require `unit > integration` and `integration >= e2e` and `unit > e2e`. If `e2e > integration` or `e2e > unit` -> ice-cream cone (CRITICAL). If `integration > unit` -> inverted pyramid (CRITICAL).
2. **Distribution** (WARNING): compute each level's share of the total. Flag if unit < 50% (too few unit), or e2e > 20% (too many E2E), against the 70/20/10 guideline.
3. **Empty levels** (WARNING): no integration tests at all, or no unit tests for a module that has domain logic.

## Step 4 — Coverage of changed code

Run real coverage when a driver is present; always do the structural mapping.

```bash
# Real coverage if PCOV or Xdebug is available
if php -m | grep -qiE 'pcov|xdebug'; then
  vendor/bin/phpunit --coverage-text --colors=never 2>/dev/null | tail -40
fi
```

Structural mapping (always): for each changed class under `src/*/Domain/` or `src/*/Application/`, check a matching test exists in the mirrored `tests/` path.

```bash
for f in $(git diff --name-only "$base"...HEAD -- 'src/**/Domain/*.php' 'src/**/Application/*.php'); do
  cls=$(basename "$f" .php)
  found=$(grep -rl "$cls" tests/ 2>/dev/null | head -1)
  [ -z "$found" ] && echo "UNTESTED: $f"
done
```

Priority: **Domain logic and use-case handlers must be unit-tested** (CRITICAL if a changed aggregate/domain-service/handler has no unit test). Thin adapters may rely on integration/functional tests.

## Step 5 — Test-double misuse

```bash
# Mocking concrete domain types instead of ports (smell)
grep -rn "createMock(\|prophesize(\|->getMockBuilder(" tests/ | grep -vE "Interface|Port|Repository|Gateway|Client|Bus|Clock|Mailer|Notifier"
# Over-mocking: many mocks in one test
grep -rc "createMock(" tests/Unit 2>/dev/null | awk -F: '$2 > 4 {print "OVER-MOCKED: "$1" ("$2" mocks)"}'
# Domain unit tests must NOT touch the DB / container
grep -rn "KernelTestCase\|EntityManager\|self::getContainer" tests/Unit/*/Domain 2>/dev/null
```

Flag: mocking a class that is not a port (CRITICAL — couples to implementation, breaks the "mock only ports" rule); Domain unit tests using the kernel/DB (CRITICAL — they must be pure PHP); >4 mocks in one test (WARNING — design smell, likely SRP violation in the unit under test).

## Step 6 — Report

```markdown
# Test Pyramid Audit — {branch} vs {base}

## Pyramid Shape
| Level | Tests | Share | Pyramid says |
|-------|-------|-------|--------------|
| Unit | u | u% | ~70% (base, most) |
| Integration | i | i% | ~20% |
| Functional + Behat (E2E) | e | e% | ~10% (top, fewest) |

Shape verdict: {HEALTHY PYRAMID} / {INVERTED — integration/E2E outnumber unit} / {ICE-CREAM CONE — E2E-heavy}

## Coverage
- Real line coverage: {x% or "no driver — structural mapping only"}
- Changed Domain/Application classes without a test: {list}

## Doubles
- Non-port mocks: {list}
- Domain unit tests touching DB/container: {list}
- Over-mocked tests (>4): {list}

## Findings
### CRITICAL
- {file}:{line} — {issue} — {fix}
### WARNING
...
### INFO
...

## Verdict
{PASS} / {CHANGES REQUIRED — n CRITICAL}
```

## Rules

- Read-only. Report only.
- A test is a test **method/scenario**, not a file — count methods.
- The pyramid percentages are guidance (WARNING); the ordering invariant and the double/coverage rules are enforced (CRITICAL).
- If `tests/` is absent or a level is empty, say so explicitly rather than failing.
- Honor `symfony-testing`: mock only ports; never mock internal domain collaborators; Domain unit tests are pure PHP; integration uses the same DB engine as production.

---
description: "Audit a git branch against every project skill — hexagonal rules, SOLID, leaky abstractions, bounded contexts, CQRS, validation, security, persistence, patterns, and tests. Produces one consolidated report with CRITICAL/WARNING/INFO findings and a verdict."
allowed-tools: Read, Glob, Grep, Bash, Task
argument-hint: "[base-branch] [-- focus instructions]"
---

# Audit Branch

Run a full architecture and quality audit on the changes in the current branch, applying every analysis skill this project ships. Read-only — never modify code.

## Input

Parse `$ARGUMENTS`:
- First token (optional) = base branch to diff against. Default: `main` (fall back to `master` if `main` is absent).
- After ` -- ` = focus instructions (e.g. "concentrate on the Order module", "only CRITICAL").

```
/audit-branch                      # diff current branch vs main
/audit-branch develop              # diff vs develop
/audit-branch main -- only security and leaky abstractions
```

## Step 1 — Determine scope

```bash
# Resolve base branch
base="${1:-main}"
git rev-parse --verify "$base" >/dev/null 2>&1 || base="master"

# Changed PHP files on this branch
git diff --name-only "$base"...HEAD -- '*.php'
git diff --stat "$base"...HEAD
```

If there are no changed PHP files, report that and stop.

## Step 2 — Run the full skill sweep on the changed files

Apply each relevant skill's detection patterns, scoped to the changed files and their modules. The skills carry the actual detection logic — consult them and run their greps against the diff scope:

| Concern | Skill | Looks for |
|---------|-------|-----------|
| Layer/boundary purity | `symfony-leaky-abstractions` | framework/infra leaking into Domain/Application, leaky ports |
| SOLID | `symfony-solid-analysis` | God classes, type switches, fat interfaces, concrete deps |
| Strategic DDD | `symfony-bounded-contexts` | cross-context coupling, foreign-entity refs, missing ACL |
| Hexagonal core rules | `symfony-hexagonal-architecture`, `symfony-ports-adapters` | dependency direction, ports/adapters, DI binding |
| Domain model | `symfony-domain-modeling` | primitive obsession, anemic entities, invariant protection |
| CQRS | `symfony-cqrs-handlers` | command/query separation, handler shape |
| Persistence | `symfony-doctrine-persistence` | raw SQL (CRITICAL), repository adapter placement |
| Validation | `symfony-validation` | 3-layer validation, DB-access constraints in Domain |
| Security | `symfony-security-voters` | endpoints missing `#[IsGranted]` |
| API shape | `symfony-api-response` | non-standard response payload |
| Async | `symfony-messenger-async` | idempotency, retry/backoff, raw cron |
| Patterns | `symfony-design-patterns` | pattern misuse / over-engineering |
| Tests | `symfony-testing` | missing or misplaced tests for changed code |

Also delegate the core-rule pass to the existing reviewer for consistency:

```
Task tool with subagent_type="hexagonal-reviewer"
prompt: "Review the diff between {base} and HEAD against the 4 core rules and additional standards. Report CRITICAL/WARNING/INFO with file:line."
```

For large diffs, you may fan out: run independent skill passes in parallel via multiple Task calls, then merge their findings.

## Step 3 — Consolidate

Deduplicate overlapping findings (e.g. a Doctrine import in Domain may surface from both leaky-abstractions and the core dependency rule — report once, cite both lenses). Group by severity, then by file.

Respect any focus instruction from `--` (narrow the skills or severities accordingly).

## Step 4 — Report

```markdown
# Branch Audit — {current-branch} vs {base}

## Summary
- Files changed: {n} PHP files
- Findings: {c} CRITICAL, {w} WARNING, {i} INFO

| Lens | CRITICAL | WARNING | INFO |
|------|----------|---------|------|
| Leaky abstractions | n | n | n |
| SOLID | n | n | n |
| Bounded contexts | n | n | n |
| Hexagonal core | n | n | n |
| Domain model | n | n | n |
| CQRS / Validation / Security / API / Async / Patterns / Tests | n | n | n |

## CRITICAL
### {file}:{line} — {lens}
- Issue: {what}
- Rule: {which skill/principle}
- Fix: {concrete remediation}

## WARNING
...

## INFO
...

## Verdict
{PASS — no CRITICAL} / {CHANGES REQUIRED — n CRITICAL must be fixed}
```

## Rules

- Read-only. Report findings; do not edit code.
- Reference `file:line` for every finding.
- Honor the project CLAUDE.md exceptions: inert `#[ORM\...]` / format `#[Assert\...]` on Domain entities are ALLOWED — do not flag them. DB-access constraints (`UniqueEntity`, `Callback`, `Expression`) in Domain and any native SQL outside migrations ARE CRITICAL.
- If a skill is not installed in the target project, skip its lens and note it in the summary rather than failing.

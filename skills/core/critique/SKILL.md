---
name: critique
description: Use when evaluating code, plans, designs, analyses, or proposed fixes for correctness, integrity, maintainability, and hidden failure modes.
---

# Skill: critique

## Purpose
Systematically find flaws in proposed solutions, code, analyses, or plans before they are acted upon.
Acts as an adversarial reviewer — not to block progress, but to ensure correctness, data integrity,
and convention compliance.

> **Project-agnostic skill.** Concrete examples in the host project may reference its own
> stack (ABP, EF Core, manager-folder conventions, etc.). The structure below is portable;
> the host project supplies the integrity checklist.

## Inputs
- Target artifact: code snippet, plan, analysis, proposed fix, or design
- Context: what problem the artifact is solving
- Optional: confidence score from the agent that produced it
- Optional: project-specific integrity checklist (tenant filter, audit columns, permission gates, etc.)

## Outputs
- Severity-ranked issue list
- Per-issue: location, impact, and suggested resolution
- Overall critique score (0.0–1.0, where 1.0 = no issues found)
- Go / No-Go recommendation with conditions

## Process

**Step 1 — Understand intent**
Read the artifact with the goal in mind. Do not critique style unless it creates ambiguity.
Only flag issues that affect correctness, data integrity, or maintainability.

**Step 2 — Apply issue taxonomy**

| Severity | Label | Meaning |
|---|---|---|
| P0 | BLOCKER | Data corruption, security bypass, irreversible operation without safeguards, broken invariant |
| P1 | CRITICAL | Wrong behavior for a common input, public-contract regression, missing data-integrity filter |
| P2 | MAJOR | Layer / convention violation, performance hot-path issue (N+1, unbounded read), missing localization for user-facing string |
| P3 | MINOR | Style, naming, or missing null-guard that does not affect current behavior |

**Step 3 — Enumerate issues**
For each issue:
```
[SEVERITY] Location (file:line if known)
What: <one sentence stating the problem>
Impact: <what breaks if unaddressed>
Fix: <concrete corrective action>
```

**Step 4 — Check integrity-critical paths**
Explicitly ask: does this artifact touch any of the project's integrity-critical surfaces?
Common categories (host project supplies the concrete list):
- Multi-tenant / data-isolation query path
- Soft-delete / logical-delete filtering
- Audit columns (created/modified/deleted metadata)
- Authorization / permission attribute
- Public response contract (DTO shape, response envelope)
- Schema migration (idempotency, backfill safety)

If yes, apply stricter scrutiny. Anything that weakens these guarantees is P1 minimum;
anything that removes them is P0.

**Step 5 — Score and recommend**
```
Critique score = 1.0 - (0.4 * P0_count + 0.25 * P1_count + 0.1 * P2_count + 0.02 * P3_count)
```
- Score >= 0.85: GO
- 0.65 <= score < 0.85: CONDITIONAL GO — list required fixes
- Score < 0.65: NO-GO — return to producing agent with full issue list

## Heuristics

- **Layer / DDD discipline.** If a higher layer instantiates a lower-layer concept without
  going through its defined factory / repository / port, that is P2 minimum.
- **Data-isolation filter.** A new query path on isolated data (per-tenant, per-user, per-org)
  that does not rely on an automatic framework filter, and lacks an explicit `WHERE` predicate
  for the isolation key, is P0.
- **Raw entity return.** If a service / handler returns the persistence entity instead of an
  explicit DTO, that is P1 — leaks navigation properties and breaks the public contract.
- **Authorization attribute.** If a new public entry point has no authorization check and no
  comment marking it as anonymous-by-design, that is P0.
- **Migration idempotency.** Any new schema migration lacking `IF EXISTS` / `IF NOT EXISTS`
  guards (or framework equivalent) is P0.
- **DTO-field rename impact.** If a renamed DTO field is consumed by client code by string
  name (JS / TS pages, mobile clients), that rename is P1 — it breaks callers silently.

## Failure Modes

| Failure | Symptom | Recovery |
|---|---|---|
| False positive storm | Minor issues inflate severity | Re-read artifact with intent in mind |
| Missing integrity check | Isolation-filter path not flagged | Run Step 4 explicitly as a separate pass |
| Critique of style over substance | P3s dominate the list | Suppress all P3 until P0/P1/P2 exhausted |
| Circular critique | Critique of the critique | Cap recursion at 1 level |

## Example (generic)

**Target:** A proposed fix for `OrderQueryHandler.GetOpenOrders` that adds a projection but
removes the existing tenant filter, on the theory that the framework's auto-filter handles it.

**Issues:**
```
[P0] OrderQueryHandler.cs:223 — tenant filter removed
What: Explicit .Where(x => x.TenantId == currentTenant) was deleted; relies on auto-filter
  which is bypassed when this method runs in a background-job context.
Impact: Cross-tenant data leak in any background-triggered execution path.
Fix: Restore the explicit filter, or, if background-job invocation is intentional, inject
  the data-isolation service explicitly and document the bypass at the call site.

[P1] OrderQueryHandler.cs:241 — projection returns persistence entity
What: .Select(x => x) instead of .Select(x => new OpenOrderDto { ... }).
Impact: Leaks navigation properties; breaks public contract; clients depending on the DTO
  shape see whatever EF loads.
Fix: Add explicit DTO projection with named fields.
```

**Score:** 0.55 — NO-GO. Return to producing agent.

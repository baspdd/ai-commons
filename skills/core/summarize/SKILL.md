---
name: summarize
description: Use when compressing completed investigations, reviews, plans, or workflow outputs into an audience-specific decision-ready briefing.
---

# Skill: summarize

## Purpose
Compress large or complex outputs into decision-ready briefings without losing critical signal.
Used as the final step in most workflows before delivering to the user.

> **Project-agnostic skill.** Host project supplies the integrity checklist for "findings
> that must always surface".

## Inputs
- Source artifact: agent output, code analysis, workflow result, or multi-step investigation
- Audience: developer | tech lead | QA | release engineer
- Target length: one-liner | paragraph | structured brief

## Outputs
- Audience-appropriate summary
- Key facts (max 5 bullets)
- Action items with owner and priority
- Open questions that block action

## Process

**Step 1 — Extract signal**
Scan the source for: decisions made, risks identified, blockers found, and next steps proposed.
Discard rationale unless the decision is controversial.

**Step 2 — Apply audience filter**

| Audience | Keep | Drop |
|---|---|---|
| developer | file paths, method names, error types, layer hints | org context |
| tech lead | architectural impact, integrity risk, timeline | low-level details |
| QA | test scenarios, expected vs. actual, repro steps | implementation details |
| release engineer | what changed, how to verify, rollback steps, migration risk | design rationale |

**Step 3 — Structure the output**
```
## Summary
<One sentence — what happened or what was found>

## Key Findings
- <finding 1>
- <finding 2>
- <finding 3> (max 5)

## Action Items
| Priority | Task | Owner |
|---|---|---|
| P0 | <task> | <agent or human> |

## Open Questions
- <question blocking action>
```

**Step 4 — Validate completeness**
Does the summary allow the audience to take the next action without reading the source?
If no: add the missing fact. If yes: done.

## Heuristics

- **Integrity findings always surface.** If a finding involves cross-isolation data exposure,
  authorization removal, audit-trail change, or non-idempotent migration, it appears in the
  summary regardless of severity — never buried in "Key Findings."
- **Confidence < 0.75 must be stated.** If the source agent flagged low confidence, the
  summary repeats it; the caller must not be misled into acting on weak evidence.
- **Action items must be specific.** "Review the code" is not an action item; "verify
  `<Method>` filters by `<isolation key>` before the projection" is.
- **Caller impact is part of the summary.** If a public method's contract changed, list
  the consumers (pages / clients / services) that depend on it. Clients that bind by string
  field name break silently when a DTO field is renamed.
- **Do not summarize an in-progress workflow.** Summarize only completed steps.

## Failure Modes

| Failure | Symptom | Recovery |
|---|---|---|
| Over-compression | Critical integrity finding omitted | Re-read source with integrity filter active |
| Under-compression | Summary longer than source | Enforce 5-bullet cap strictly |
| Wrong audience | Developer summary delivered to QA | Re-apply Step 2 audience filter |
| Missing action items | Summary ends with findings, no next step | Always produce at least one action item |

## Example (generic)

**Source:** 800-word research output on a Pending-tab dashboard showing stale Done items
after a refactor.

**Summary (developer audience):**
```
## Summary
The close-flow refactor removed a `Status != Closed` filter from
`DashboardService.GetPendingItems`, so Done items now leak into the Pending tab.

## Key Findings
- `DashboardService.GetPendingItems` (line ~210) no longer filters by Status.
- Pre-refactor commit `<hash>` contained the filter; the consolidated method dropped it.
- `DashboardRepository.GetPendingQueryable()` is unchanged — the bug lives in the service projection.
- No isolation-filter regression — tenant scope is intact.

## Action Items
| Priority | Task | Owner |
|---|---|---|
| P0 | Re-add Status filter (Status != Done && != Closed) before .Select(...) | developer |
| P1 | Add regression test covering Done-item exclusion | developer (or skip if test suite is light) |
| P2 | Add comment near the filter explaining why the close-flow refactor must preserve it | developer |

## Open Questions
- Should the Pending tab also exclude items with Status = Rejected? (Ask PM)
```

**Confidence: 0.84**

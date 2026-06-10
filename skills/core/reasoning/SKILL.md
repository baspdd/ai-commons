---
name: reasoning
description: Use when a problem is complex, crosses architectural layers, contains uncertain assumptions, or must be decomposed into evidence-backed sub-questions.
---

# Skill: reasoning

## Purpose
Decompose complex problems into verifiable sub-questions before attempting answers.
Prevents premature conclusions. Mandates evidence grounding over inference leaps.

> **Project-agnostic skill.** Examples in host projects may name concrete layers, modules,
> or persistence paths. The decomposition method below is portable.

## Inputs
- Problem statement (free text)
- Optional: relevant file paths or module names
- Optional: prior context from memory/

## Outputs
- Structured decomposition tree
- List of open questions requiring investigation
- Dependency order for resolving sub-questions
- Confidence score per sub-question

## Process

**Step 1 — Restate the problem**
Paraphrase in one sentence what is being asked. Identify the domain (which layer, module,
persistence path). This surfaces hidden assumptions in the original request.

**Step 2 — Identify what is known vs. unknown**
Split facts into:
- KNOWN: derivable from reading code, config, or migrations right now
- UNKNOWN: requires investigation (caller trace, query inspection, runtime behavior)

**Step 3 — Decompose into atomic sub-questions**
Each sub-question must be answerable independently. If two questions share a prerequisite,
draw an explicit dependency edge. Maximum depth: 4 levels.

**Step 4 — Order by dependency**
Resolve leaf questions first. Never answer a parent question before its children are resolved.

**Step 5 — Score each branch**
Assign confidence (0.0–1.0) to each answer as it is resolved. Propagate uncertainty upward:
the parent score is the minimum of its children's scores.

**Step 6 — Synthesize**
Assemble resolved sub-answers into a coherent top-level answer.
State which open questions remain and what their impact on confidence is.

## Heuristics

- **Layer discipline.** A question about why something fails that crosses architectural
  layers is at least two questions. Don't conflate "the API returned wrong data" with
  "the data query is wrong" — split them.
- **Module boundary.** Trace failures by module / bounded context. If the symptom spans
  modules, decompose per module.
- **Persistence path resolution.** A question about wrong data must trace:
  service → repository / DAO → data source (ORM query or raw SQL).
  Stop at the layer where evidence lives, not where the symptom appeared.
- **Isolation context.** A question about "user sees wrong data" has at least two distinct
  flavors: cross-isolation leak (tenant / org / workspace filter missing) vs. within-isolation
  authorization mismatch (role / scope). Treat them as separate sub-questions.
- **Delete vs. status.** "Deleted row still appears" can be either missing soft-delete filter
  OR a status enum (Closed / Archived / Done, etc.) not excluded. Don't assume which —
  verify with code reading.

## Failure Modes

| Failure | Symptom | Recovery |
|---|---|---|
| Premature synthesis | Answer given before all leaf questions resolved | Force return to Step 4 |
| Circular dependency | Sub-question A requires B which requires A | Break with assumption; flag assumption explicitly |
| Scope creep | Sub-questions multiply beyond 12 nodes | Prune to most impactful 6; defer rest |
| Over-confidence | Score inflated without file evidence | Require cite before score > 0.7 |
| Layer-blur | Single question spans multiple layers | Split into per-layer sub-questions |

## Example (generic)

**Input:** "After the close-flow refactor, items marked as Done are still listed in the
Pending tab of the dashboard."

**Decomposition:**
```
Root: why are Done items still listed in the Pending tab after the refactor?
├── [KNOWN] Which service method serves the Pending tab? → DashboardService.GetPendingItems
├── [UNKNOWN] What did the refactor change in that method? → git diff vs. previous commit
│   └── Was a Status / IsDone filter removed? → diff inspection
├── [UNKNOWN] Does the query rely on a raw SQL function?
│   └── If yes, was the function modified? → grep migration files for changes
├── [UNKNOWN] Is the issue actually a cache problem on the frontend?
│   └── Does the page re-fetch on mark-done? → read the page module
└── [UNKNOWN] Is soft-delete vs. closed-status being confused?
    └── Read the entity — IsDeleted vs. Status fields
```

**Confidence:** 0.55 (three branches unresolved). Escalate unknowns to research_agent.

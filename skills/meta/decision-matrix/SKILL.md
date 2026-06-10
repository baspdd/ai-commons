---
name: decision-matrix
description: Use when comparing two or more viable engineering options with explicit tradeoffs, constraints, risks, and weighted decision criteria.
---

# Skill: decision_matrix

## Purpose
Evaluate competing options using a weighted, scored comparison — replacing gut-feel
decisions with transparent, reproducible reasoning.

> **Project-agnostic skill.** Host projects override the default dimensions / weights
> to match their own integrity profile (multi-tenant, safety-critical, regulated, etc.).

## Inputs
- Decision to make (free text)
- Options: list of 2–5 alternatives
- Optional: constraints (performance, time, public-API stability, isolation boundary)
- Optional: weights for evaluation dimensions

## Outputs
- Scored decision matrix (table)
- Recommended option with justification
- Risk summary for the recommended option
- Conditions under which the second-best option becomes preferable

## Process

**Step 1 — Define dimensions**
Select 4–6 evaluation dimensions relevant to the decision.
Default dimensions for engineering decisions:

| Dimension | Weight | Description |
|---|---|---|
| Correctness | 0.25 | Does the option solve the problem reliably for all inputs? |
| Data Integrity | 0.25 | Isolation boundary, soft-delete, audit trail, authorization preserved? |
| Convention Alignment | 0.20 | Fits the existing layer / module / DTO / response shape conventions? |
| Effort | 0.15 | Time and complexity to implement |
| Testability | 0.10 | Can it be verified by build + manual smoke without prod data? |
| Reversibility | 0.05 | If wrong, how easily can we roll back? |

Weights must sum to 1.0. Adjust per decision context (host project may rename or reweight).

**Step 2 — Score each option**
Score each option per dimension on a 1–5 scale:

| Score | Meaning |
|---|---|
| 5 | Excellent — no concerns |
| 4 | Good — minor concerns |
| 3 | Acceptable — notable trade-off |
| 2 | Poor — significant weakness |
| 1 | Unacceptable — disqualifying |

**Step 3 — Compute weighted scores**
```
Option_score = Σ (dimension_score × dimension_weight)
```

**Step 4 — Build the matrix**
```markdown
| Option | Correctness (0.25) | Integrity (0.25) | Convention (0.20) | Effort (0.15) | Testability (0.10) | Reversibility (0.05) | Total |
|---|---|---|---|---|---|---|---|
| Option A | 4 | 5 | 5 | 3 | 4 | 4 | 4.20 |
| Option B | 4 | 3 | 3 | 5 | 5 | 5 | 3.85 |
```

**Step 5 — Recommend and justify**
State the winner. Explain the deciding factor. Note the margin.
If margin < 0.15: options are effectively tied — add a tiebreaker dimension.
If Data Integrity score is 1 on any option: that option is disqualified regardless of total.

**Step 6 — Conditions for second option**
"Option B becomes preferable if: <specific condition, e.g., 'feature must ship this week
and follow-up integrity migration is acceptable'>."

## Heuristics

- **Data Integrity weight must be >= 0.20** for any decision touching isolation queries,
  audit columns, authorization gates, or migration files.
- Never recommend an option with Data Integrity score <= 2 for a public API surface or
  any isolation-scoped data path, regardless of total score.
- Effort score of 1 (unacceptable complexity) should trigger a question:
  "Is there a simpler framing of this decision?"
- If all options score <= 3 on Correctness: the decision is premature.
  Return to research before proceeding.
- **Convention Alignment is not a tiebreaker — it is a primary axis.** A new pattern that
  diverges from existing module conventions costs the team velocity for years.

## Failure Modes

| Failure | Symptom | Recovery |
|---|---|---|
| Dimension gaming | Weights manipulated to favor a pre-chosen option | Have a second agent (critique skill) review the weights |
| Missing option | Obvious alternative not considered | Ask: "Is there a do-nothing option? A hybrid?" |
| Score inflation | Everything scores 4–5 | Force stack-ranking: at least one dimension per option must differ |
| Ignoring constraints | Winning option breaks public DTO contract | Add public-contract preservation as a disqualifying filter |

## Example (generic)

**Decision:** How to model a `Phase` attribute on an `Item` — enum int column vs. FK to a
lookup table vs. JSONB / JSON column.

**Dimensions (default):**
- Correctness (0.25), Data Integrity (0.25), Convention Alignment (0.20), Effort (0.15),
  Testability (0.10), Reversibility (0.05)

| Option | Correctness | Integrity | Convention | Effort | Testability | Reversibility | Total |
|---|---|---|---|---|---|---|---|
| Enum int column | 5 | 5 | 5 | 5 | 5 | 3 | **4.85** |
| FK to lookup table | 4 | 5 | 4 | 3 | 4 | 4 | 4.15 |
| JSON / JSONB column | 3 | 3 | 2 | 4 | 3 | 4 | 2.95 |

**Recommendation:** Enum int column (total: 4.85).
Consistent with existing status fields elsewhere in the codebase. Display values handled
via the existing localization layer. Migration is a single column-add with audit columns
preserved. Margin over FK option is 0.70 — clear winner.

**Conditions for FK option:** If phases must be admin-configurable at runtime (new phases
added without a migration). Not the case here — phase set is fixed by spec.

**Risk note:** Reversibility scored 3 because a future migration to FK requires data
conversion. If phases later need user-defined values, plan the FK migration in advance.

# Request Workflow

> **Always active.** This is the canonical shared request lifecycle. It governs *how the
> assistant responds to every request that reads, changes, investigates, plans, or reviews
> code or configuration*, regardless of language or repository. It is referenced from
> `CLAUDE.md` so it loads on every session, and it wraps — and runs *before* — the code rule
> files selected through `RULE_ROUTER.md`
> (`AI_PROMPT_BACKEND/FRONTEND/SQL.md`, `REFACTOR.md`, …).
>
> A project profile (`.ai/PROJECT.md`) may add stricter local guardrails, but must not relax
> the rules below.

## Non-Negotiable Rules

- Do not classify work from prompt length, wording, line count, or apparent file count.
- Ground behavioral claims in actual code before planning or editing.
- For every request that may modify files, complete the lightweight impact scan before
  assigning `DIRECT`, `SCOPED`, or `FULL`.
- Emit `Workflow: DIRECT|SCOPED|FULL - <evidence-based reason>` after classification.
- Announce each skill or subagent when it is actually used; do not claim implicit calls.
- The primary agent is the only writer.

Read-only questions may still need broad research, but they skip the implementation prompt,
approval, `load-rules`, and `compliance-check` when no files are edited.

## 1. Provisional Triage

Identify whether the request is read-only or mutating and classify its intent as feature,
bug, refactor, investigation, review, or project configuration. This is provisional only;
do not select a workflow level yet.

For a mutating request, inspect enough of the real execution path to establish:

1. Entry point and observable behavior.
2. Direct callers, consumers, and downstream execution paths.
3. Affected module and architectural layers.
4. Data access, public contracts, permissions, tenant/soft-delete, audit, migration,
   history, notification, background-job, and report surfaces.
5. Existing peer pattern, verification route, uncertainties, and open decisions.

When CodeGraph is available and indexed, use its context/impact tools first when they cover
the code shape; otherwise use targeted file reads and search. The scan is evidence gathering,
not implementation.

For a read-only request, perform the same targeted evidence scan at the breadth needed by
the question, then classify it with the gate below. A local explanation is normally
`DIRECT`; a bounded multi-file investigation is `SCOPED`; a cross-feature or uncertain
investigation is `FULL`. Skip every mutation-only step afterward.

## 2. Impact Gate

Apply the following order after the scan.

### FULL - any hard trigger

Classify as `FULL` when any item is true:

- New or changed business rule, workflow state, or state transition.
- More than one feature, module, or independently observable execution path.
- Permission/authorization, tenant isolation, soft-delete, audit contract, schema,
  migration, data backfill, or public API/DTO/response-shape impact.
- History, notification, background-job, integration, or report behavior may change.
- A product or architecture decision remains open.
- Root cause, caller impact, acceptance criteria, or verification path remains uncertain.

### DIRECT - all conditions required

Classify as `DIRECT` only when every item is true:

- The requested behavior is local, unambiguous, and follows an existing peer pattern.
- The affected execution path and its callers/consumers are closed and understood.
- No hard trigger or open decision exists.
- Verification is local, deterministic, and low-risk.

### SCOPED - the remaining cases

Classify as `SCOPED` when the change is cohesive and bounded to one feature or UI surface,
may require several related files, and the scan closes all hard-risk and uncertainty paths,
but the work is not trivial enough for `DIRECT`.

Examples:

- A long design request that changes one UI surface (markup, styles, client script) while
  preserving its API and business behavior is normally `SCOPED`.
- A short request to change an approval or business behavior is `FULL` when it reaches
  status, permission, history, notifications, or reports.
- A known local rename or display correction is `DIRECT`.

## 3. Execution by Level

### DIRECT

1. Read the exact affected code.
2. If editing rule-bound files, invoke `load-rules` before the first edit.
3. Implement directly with no formal prompt, plan, or subagent.
4. Run the smallest relevant verification and invoke `compliance-check` after edits.

### SCOPED

1. Show a short impact summary: behavior, bounded scope, risks cleared, and verification.
2. Create a short implementation checklist and continue; the original request authorizes
   implementation, so do not wait for separate plan approval.
3. Keep the primary agent as the sole writer and load rules before each new file category.
4. Run targeted verification and `compliance-check`.
5. Use a read-only reviewer only for an evidence gap or genuinely mixed file scope.

### FULL

For a read-only investigation or review:

1. Build the context/impact map with CodeGraph when available or targeted reads/search
   otherwise; use `reasoning` and `workflow-planner` to structure the analysis.
2. Run only the relevant read-only chain (research / feature analysis / bug triage /
   refactor planning). Add an integrity assessment when an integrity trigger exists.
3. Return the evidence report, unresolved questions, verification advice, and confidence.
   Do not create an implementation prompt, request plan approval, invoke `load-rules`, or
   run `compliance-check` unless the user subsequently requests a mutation.

For a mutating request, run the full sequence:

1. Decompose the request and select the read-only analysis chain and artifact handoffs
   (use `reasoning` and `workflow-planner` when available).
2. Build the initial context/impact map with CodeGraph when indexed, or targeted
   reads/search otherwise, then route by task type using the project's read-only agents:
   - Feature: research -> feature analysis.
   - Bug: bug triage -> research.
   - Refactor: refactor planning plus caller-impact analysis.
   - Integrity-sensitive: add an integrity assessment.
3. Produce a grounded implementation prompt from the impact report (use `prompt-optimizer`
   when available). Use `decision-matrix` only when two or more viable choices remain.
4. Produce a decision-complete implementation plan and apply `critique` before presenting
   it. Then stop and wait for explicit user approval. Tool, command, or sandbox approval
   does not approve the plan.
5. After approval, keep one writer, invoke `load-rules`, and implement the approved scope.
6. Run relevant tests/builds, then `compliance-check`.
7. Run an independent reviewer appropriate to the changed scope; repeat the integrity
   assessment against the actual diff when an integrity trigger exists.
8. Fix findings and repeat affected verification before summarizing and reporting
   completion.

## 4. Reclassification

Re-evaluate when new callers, consumers, contracts, features, or integrity surfaces appear.
If `DIRECT` or `SCOPED` expands into a `FULL` trigger:

1. Stop at a safe boundary without discarding existing user work.
2. Report completed edits and the newly discovered impact.
3. Produce the `FULL` impact report, implementation prompt, and plan.
4. Wait for explicit approval before any further implementation.

Do not silently continue under the earlier classification.

## 5. Completion Evidence

Report the final workflow level, files changed, rules/skills/agents actually used, tests and
builds run, review verdicts, skipped checks, and remaining uncertainty. Never report done
while compliance or required review is blocked.

---
name: workflow-planner
description: Use when a task needs selection and ordering of multiple agents or workflows, explicit artifact handoffs, confidence targets, or safe parallelization.
---

# Skill: workflow_planner

## Purpose
Select and configure the right agent sequence for a given task.
Acts as the routing layer between raw user intent and structured multi-agent execution.

> **Project-agnostic skill.** Host projects supply the actual agent roster and the
> integrity-gate trigger surfaces.

## Inputs
- Task description (free text)
- Optional: urgency (P0 hotfix | normal | exploratory)
- Optional: available agents (if operating in restricted context)

## Outputs
- Selected workflow (with justification)
- Agent sequence with inputs / outputs per step
- Estimated confidence achievable at completion
- Risks and mitigation per step
- Parallel execution opportunities

## Process

**Step 1 — Classify the task**

| Task type | Trigger keywords | Recommended chain |
|---|---|---|
| New feature | "add", "implement", "build", "new" | feature_analyst → integrity_assessment → research → review |
| Bug fix | "broken", "error", "failing", "wrong", "not working" | bug_hunter → research → review |
| Code quality | "clean up", "refactor", "improve", "simplify" | refactor → review (integrity-focused if touches data path) |
| Integrity gate | "safe to merge?", "isolation leak?", "permission impact?" | integrity_assessment (standalone) |
| Architecture question | "how does", "why does", "where is" | research (standalone) → summarize |

**Step 2 — Compose the chain**
Load the selected chain. Verify all listed agents are declared in the host project's
`config.yaml`. If an agent is unavailable, substitute with the nearest capable agent and
note the gap.

**Step 3 — Specify per-step inputs**
For each step in the chain, define what it receives from the previous step.
Intermediate state is represented as a named artifact:

```yaml
step_1:
  agent: bug_hunter_agent
  input: symptom_description
  output_artifact: bug_report

step_2:
  agent: research_agent
  input: bug_report
  output_artifact: root_cause_analysis

step_3:
  agent: review_agent
  input: proposed_fix + root_cause_analysis
  output_artifact: review_verdict
```

**Step 4 — Identify parallel opportunities**
Steps with no dependency between them can run in parallel.
Examples:
- review_agent (code) + integrity_assessment_agent (data paths) can both start after
  research_agent produces the root cause.
- For a new feature: feature_analyst output can be fed in parallel to integrity_assessment
  (for authorization + migration check) and to research_agent (for caller-impact mapping).

**Step 5 — Set confidence expectations**
Estimate final confidence based on task type:
- Bug resolution with clear regression (recent commit + clear symptom): 0.80+
- Feature scope with full spec + no open decisions: 0.80+
- Refactor with caller trace complete: 0.75+
- Exploratory analysis: 0.55–0.70
- Migration / data-path change without lab verification: 0.60 max

## Heuristics

- **Integrity gate is mandatory for data-layer changes.** Any task touching isolation
  filtering, soft-delete, audit columns, authorization attributes, or migration files
  must include integrity_assessment_agent before review_agent.
- **P0 hotfixes skip feature_analyst** — go directly: bug_hunter → research → review.
- **Exploratory tasks** ("what's going on with X?") should run research_agent first,
  then decide next chain based on findings — do not over-plan upfront.
- **Multi-module scope split.** If the task spans more than 2 modules / bounded contexts,
  the orchestrator should split into independent sub-workflows per module rather than
  one long chain.
- **Frontend-only changes** can skip integrity_assessment unless they break the public
  response contract or rename a DTO field consumed by other pages.

## Failure Modes

| Failure | Symptom | Recovery |
|---|---|---|
| Wrong chain selected | Agents produce irrelevant output | Re-classify task at Step 1 |
| Missing step dependency | Agent receives incomplete context | Add explicit artifact passing |
| Over-planned chain | 8-step chain for a 2-line fix | Apply minimum viable chain |
| Parallel race condition | Two agents modify same artifact | Serialize those specific steps |
| Skipped integrity gate | Isolation/audit/permission change merged without check | Add integrity_assessment_agent as mandatory gate |

## Example (generic)

**Input:** "After the close-flow refactor, Done items still appear on the Pending tab. Fix it."

**Classification:** Bug fix (regression after recent change).
**Chain:** bug_hunter → research → review (integrity gate optional — read-only query).

**Step sequence:**
```
1. bug_hunter_agent
   Input:  symptom — "Done items visible on Pending tab after refactor <commit hash>"
   Output: bug_report (file paths, hypothesis ranking, integrity-flag: CLEAR)

2. research_agent
   Input:  bug_report
   Output: root_cause_analysis (diff inspection, exact filter removed, fix proposal)

3. review_agent (depth: standard)
   Input:  proposed fix + root_cause_analysis
   Output: review_verdict (GO / CONDITIONAL / NO-GO + change requests)
```

**Parallel opportunity:** None — sequential dependencies.

**Confidence target:** 0.80 (regression with named commit, low ambiguity).

**Integrity gate:** Not required — the fix re-adds a Status filter; no isolation / audit /
authorization surface. Re-evaluate if research_agent finds the fix also touches the
projection shape or repository query (then add integrity_assessment_agent).

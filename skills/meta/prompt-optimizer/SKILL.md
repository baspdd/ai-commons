---
name: prompt-optimizer
description: Use when a prompt is vague, repeatedly underperforms, selects the wrong scope, lacks evidence requirements, or needs a precise output contract.
---

# Skill: prompt_optimizer

## Purpose
Rewrite vague or underperforming prompts into precise, high-yield instructions
that produce consistent, actionable outputs from AI agents.

> **Project-agnostic skill.** Concrete examples in host projects substitute project-specific
> file paths, rule files, and module conventions.

## Inputs
- Original prompt (raw user request or agent instruction)
- Target agent or skill
- Desired output format (optional)
- Known failure mode of the original prompt (optional)

## Outputs
- Optimized prompt (drop-in replacement)
- Explanation of changes (diff of intent, not words)
- Confidence that the optimized version will produce better output

## Process

**Step 1 — Diagnose the original prompt**
Identify failure patterns:

| Pattern | Symptom | Fix |
|---|---|---|
| Underspecified | Agent asks clarifying questions | Add scope, constraints, expected format |
| Overspecified | Agent ignores context to follow script | Remove procedural steps; keep goals |
| Ambiguous target | Agent picks wrong service / module | Name the exact file or class |
| Missing output contract | Agent produces whatever format it prefers | Specify exact output schema |
| No grounding instruction | Agent hallucinates types / methods that don't exist | Add "read the file before answering" |
| Missing integrity framing | Agent omits isolation / audit / authorization risk flags | Add "flag any paths that touch <project's integrity surfaces>" |
| Missing rule-file load | Agent writes code that violates rules | Add "load <rule file path> first" |

**Step 2 — Apply optimization patterns**

**Pattern A — Anchor to evidence:**
Before: "Review the OrderService."
After: "Read `OrderService.cs` in full. Apply the code_review skill checklist —
architectural layering, data integrity, public-contract preservation. Produce findings in
[SEVERITY] File:line — What — Impact — Fix format."

**Pattern B — Constrain scope:**
Before: "What's wrong with the Pending tab?"
After: "Investigate why Done items still appear on the Pending tab after the close-flow
refactor. Scope: `<service file>` and its repository. Start by diffing the method against
commit `<hash>`."

**Pattern C — Specify output schema:**
Before: "Summarize the bug."
After: "Produce a bug report using the schema in bug_report.md. Fill every section.
Include file paths with line numbers. State confidence score at the end."

**Pattern D — Add agent persona:**
Before: "Check the integrity."
After: "You are the integrity assessment agent. Apply the host project's integrity passes
(isolation, soft-delete, audit, authorization, public contract). Output: risk matrix +
verdict (APPROVED / CONDITIONAL / BLOCKED) + verification checklist."

**Pattern E — Add failure handling:**
Before: "Analyze the close-flow."
After: "Analyze `<service>.cs`. If you cannot determine root cause with confidence > 0.65,
state the open questions and escalate to research_agent rather than guessing."

**Pattern F — Bind to rule files:**
Before: "Write a new page for X."
After: "Before writing any frontend code, load the project's frontend rule file. Apply
the host conventions (e.g. jQuery-first + cached selectors + concise Logic comments,
or the framework's own). Output: page module following an existing peer file's conventions
(cite one explicitly)."

**Step 3 — Validate the optimized prompt**
Check:
- Can an agent follow it without asking for clarification?
- Does it specify what to read (grounding)?
- Does it specify what to produce (output contract)?
- Does it bound the scope (no open-ended "analyze everything")?
- Does it surface integrity / authorization / migration risks if applicable?

## Heuristics

- The single most effective optimization is adding a grounding instruction:
  "Read X before answering." Eliminates most hallucination about project-specific types
  and method names.
- Never optimize for brevity at the expense of output format specification.
  A longer prompt that produces structured output beats a short prompt that produces prose.
- Integrity prompts: always add "flag any paths that touch <project's integrity surfaces>"
  as a standing instruction for any analysis of public APIs or migrations.
- For frontend work: always cite an existing peer file (e.g., an existing page module) to
  anchor the agent against the project's frontend pattern.

## Failure Modes

| Failure | Symptom | Recovery |
|---|---|---|
| Over-engineered prompt | Agent follows script rigidly, misses edge cases | Remove numbered steps; use goals + constraints |
| Prompt too long | Agent satisfices early parts, ignores late parts | Put the most important instruction first |
| Conflicting constraints | Agent paralyzed by impossible requirements | Prioritize: integrity > correctness > convention > format |
| Missing convention reference | Agent invents a new pattern divergent from module | Always cite an existing peer file as the model |

## Example (generic)

**Original:** "Look at the Pending tab and tell me what's broken."

**Optimized:**
```
Read these files in full:
- <path to the service handling the Pending tab>
- <path to its repository>
- <path to the page module rendering the tab>

Question: after the merge of `<commit hash>` (close-flow refactor), Done items still
appear on the Pending tab. Why?

Investigate in this order:
1. Diff the Pending-tab query method against the pre-merge version.
2. Identify any removed filter — Status, IsDeleted, IsCompleted.
3. Trace the client code to confirm it re-fetches after mark-done (rule out client cache).
4. Flag any isolation-filter or audit-column regression you encounter.

Produce findings in:
[P0|P1|P2|P3] File:line — What is wrong — Impact — Suggested fix

End with: overall confidence score (0.0–1.0) and any open questions blocking action.
```

**Changes:** Added grounding (3 specific files), scoped the investigation, ordered the
diagnostic steps, surfaced integrity concerns as a standing flag, specified output format,
and requested a confidence score.

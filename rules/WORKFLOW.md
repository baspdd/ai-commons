# Request Workflow Rules (Language-Agnostic)

> **Always active.** This file governs *how Claude responds to every user request*,
> regardless of language or repo. It is referenced from the global `CLAUDE.md` so it
> loads on every session. It wraps — and runs *before* — the code rule files
> (`AI_PROMPT_BACKEND/FRONTEND/SQL.md`, `REFACTOR.md`, …).

---

## The Complexity Gate (decide this FIRST)

Before doing anything, classify the request:

| Class | Examples | What to do |
|---|---|---|
| **Simple** | Single-line change, rename, trivial fix, answer a direct question, read/explain code, one obvious edit in one file | **Skip Step 1 + Step 2.** Do it / answer directly. No prompt restatement, no todo list. |
| **Complex** | Multi-step, multi-file, a new feature, a refactor, anything spanning backend + frontend + SQL, anything with >1 distinct sub-task or non-obvious scope | Run **all 3 steps** below in order. |

When unsure, treat it as **complex** — the cost of a short prompt + todo list is far
lower than the cost of building the wrong thing.

---

## The 3 Steps (complex tasks only)

### Step 1 — Write the prompt

Restate the user's request as one precise, self-contained prompt before touching code:

- **Goal** — one sentence: what done looks like.
- **Scope** — exact files / classes / modules in play; what is explicitly out of scope.
- **Constraints** — rule files that apply, public-contract / audit / tenant guardrails, behavior to preserve.
- **Output** — what will be produced (code changes, migration, test, doc).

Present this prompt to the user, then **proceed** (do not block waiting for approval
unless the request is genuinely ambiguous — if ambiguous, ask a clarifying question
instead of guessing). The user can correct course after seeing the prompt.

### Step 2 — Todo list

Create a `TodoWrite` list that breaks the prompt into ordered, verifiable items.
One item per distinct unit of work. Keep exactly one item `in_progress` at a time;
mark each `completed` immediately when its verification passes.

### Step 3 — Changes

Execute the todo list. Inside this step the existing mandatory rules still apply:

1. **Load rule files first** — per the global `CLAUDE.md` rule-loading table, before the
   first edit/create of any `.cs` / `.js` / `.ts` / `.cshtml` / `.sql` file.
2. Implement following the loaded language rule file(s).
3. **Compliance self-check** — review each change against the loaded rule file and fix
   violations *before* reporting done (per global `CLAUDE.md`).
4. State outcomes faithfully (tests run, what passed, what was skipped).

---

## Anti-Patterns

| Don't | Do |
|---|---|
| Run all 3 steps for a one-line fix | Apply the complexity gate — simple → just do it |
| Skip Step 1 then build the wrong thing | Restate scope first on anything non-trivial |
| Write a todo list of 1 item | If it's 1 item, it was a simple task — skip Step 2 |
| Block forever on "approve the prompt?" | Present prompt, proceed; only stop if truly ambiguous |
| Treat Step 3 as exempt from rule files | Rule-loading + compliance check still mandatory inside Step 3 |

---

**End of workflow rule file**

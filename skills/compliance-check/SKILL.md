---
name: compliance-check
description: Use after editing .cs, .js, .ts, .cshtml, or .sql files and before reporting completion, committing, or opening a pull request.
---

# compliance-check

This skill enforces the canonical "self-review before done" policy. It runs against files
**you** just edited in this session, not arbitrary scope.

## When to invoke

- Immediately after the last edit/create of a file in a category below, and before reporting the task as done:
  - `*.cs` / `*.js` / `*.ts` / `*.cshtml` / `*.sql`
- When the user asks "did you check the rules?" or "are you done?"
- Before opening a PR or committing on the user's behalf

Skip when no rule-bound files were edited this session.

## Inputs

- The list of files modified this session (from your own memory of edits; if uncertain, `git status` + `git diff --name-only`).

## Procedure

1. **Confirm rule files are loaded.** If you cannot recall reading the matching rule files,
   load them now via the `load-rules` skill or direct file reads.

2. **For each edited file, run the matching checklist below.** Don't skip steps; don't claim "looks good" without naming the checks.

3. **Fix every violation in place.** Do not surface the violations as a to-do for the user — that defeats the purpose of self-review.

4. **Report.** Output a short table of files vs. status. Only after every file is **PASS** can you report the task as done.

## Checklist — Frontend (`*.js` / `*.ts` / `*.cshtml`)

Source: `rules/AI_PROMPT_FRONTEND.md`

- [ ] Function comments follow `AI_PROMPT_FRONTEND.md`: concise `Logic:` comments by default, full `@param` / `@returns` JSDoc only for non-obvious shared/public API contracts
- [ ] Inline numbered comments (`// 1. Validate`, `// 2. Parse`, …) at major steps inside the body
- [ ] **No single-letter / abbreviated variables anywhere** — `e` → `event`, `err` → `error`, `i` → `index`, `d` → `detail`, `t` → `task`, `f` → `file` (applies to `catch` clauses and arrow params)
- [ ] **jQuery-first** — `$('#id')` not `document.getElementById`, `$el.val()` not `el.value`, `$el.on('click', …)` not `addEventListener`
- [ ] Cached `$selector` vars declared at top + assigned in `cacheSelectors()` for page-level JS; no repeated `$('#id')` in render methods
- [ ] `===` / `!==` not `==` / `!=` unless coercion is intentional **and** commented
- [ ] Section banners use em-dash `// ─── SECTION NAME ──` for page-level JS
- [ ] Flat `renderXxx()` / `onXxxClick()` functions, not object-based pseudo-classes
- [ ] No premature extraction — short single-use logic is inlined

## Checklist — Backend (`*.cs`)

Source: `rules/AI_PROMPT_BACKEND.md`

- [ ] Application Service owns authorization, use-case validation, orchestration, and public mapping
- [ ] Domain entity/manager/service owns reusable business invariants and state transitions
- [ ] Repository owns only persistence/query behavior; no permission, workflow, or API-envelope logic
- [ ] Existing public contract remains stable: typed DTO, `ApiResult`, `ServiceResult`,
      `PagedResultDto<T>`, or legacy `{ isSuccess, data, msg }`
- [ ] Repository methods return typed persistence/query data rather than public response envelopes
- [ ] Input validation is placed at the layer that owns the rule
- [ ] Unexpected exceptions are handled by ABP or translated once at the Application boundary
- [ ] No N+1 — batched via `Where(ids.Contains(...))` + `ToDictionaryAsync()`
- [ ] `.Select(...)` projection before `.ToListAsync()` — no raw entity returns
- [ ] `.AsNoTracking()` on read-only queries; all DB calls `async` (no `.Result` / `.Wait()`)
- [ ] Raw SQL parameterized via `NpgsqlParameter`
- [ ] Audit fields match the entity's actual base class/interfaces
- [ ] ABP tenant/soft-delete filters are verified; raw SQL or disabled filters scope explicitly
- [ ] Regions and XML docs match sibling conventions instead of being added mechanically

## Checklist — SQL (`*.sql`)

Source: `rules/AI_PROMPT_SQL.md`

- [ ] Identifiers quoted PascalCase, tables prefixed `WM_`
- [ ] plpgsql locals `snake_case`; parameters `_param` / `p_param`
- [ ] `DROP IF EXISTS` before `CREATE` (idempotent)
- [ ] Audit columns match the entity base class/interfaces
- [ ] `OWNER TO dev_wm` set on tables and functions
- [ ] `WHERE "IsDeleted" = false` on reads of soft-delete entities unless intentionally bypassed
- [ ] CTEs over deep subqueries; each CTE commented
- [ ] Migration filename matches convention; header comment with version / author / ticket

## Checklist — All files

- [ ] Matches existing file conventions (indent, quote style, import order) when rules are silent
- [ ] No backward-compat shims, no commented-out leftovers, no unused imports
- [ ] Comments are English (per repo convention)
- [ ] No emojis unless the user asked for them

## Output

```
## Compliance check

| File | Status | Notes |
|---|---|---|
| <relative path> | PASS / FIXED | <if FIXED: 1-line summary of what was fixed> |
| ... | ... | ... |

Done. <one sentence: ready for user / what's still in progress>
```

Status meanings:
- **PASS** — no violations found
- **FIXED** — violations found and corrected in this session (must list what was fixed)

If you cannot fix a violation (e.g. it requires a decision from the user), call out the file as **BLOCKED** with a one-line question and stop.

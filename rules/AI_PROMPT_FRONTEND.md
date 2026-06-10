# AI Prompt: Frontend (JavaScript / TypeScript) Rules

> **For refactoring:** Combine with `REFACTOR.md` which contains the general refactoring process.

## Work Management Frontend Defaults

This file is the canonical frontend rule source for Work Management JavaScript,
TypeScript, Razor script blocks, jQuery, and DevExtreme UI work. Do not duplicate
frontend/comment/extract rules in project memory or Codex summary files.

- Keep code clean, compact, and readable.
- Preserve existing comments during edits and refactors.
- Prefer this declaration order in page-level scripts: constants, `let`/`var` state, init, helpers, main functions, child functions immediately below their parent.
- Inline simple linear logic, even when the parent function becomes longer.
- Extract only complex logic or shared logic used by 2+ callers.
- Place child helpers used by one parent immediately below that parent; lift helpers used by 2+ callers into the shared helpers section.
- Use concise logic-only comments by default: `/** Logic: ... */` or `// Logic: ...`.
- Use full `@param` / `@returns` JSDoc only for non-obvious shared/public API contracts.
- Prefer early returns to reduce deep nesting.

## JavaScript/TypeScript Rules

**Function Structure**
- Default function comment style is a short logic comment, not full JSDoc:
  ```js
  /**
   * Logic:
   * - Key behavior or reason this function exists
   * - Important edge case only if it is not obvious from code
   */
  ```
- Use a one-line `// Logic: ...` comment for short single-step helpers.
- Full JSDoc with `@param` / `@returns` is an exception, not the default:
  ```js
  /**
   * Brief description
   *
   * @param {type} paramName - description
   * @returns {type} - description
   *
   * Logic:
   * - Bullet list explaining key steps
   * - Edge cases (empty input, null handling)
   */
  ```
- Naming: `camelCase` for functions/vars, `PascalCase` for classes/components
- Error handling: `try/catch` or early returns for validation
- Constants: `UPPER_SNAKE_CASE` at top of file or in separate config

**Comments & Logic Documentation (must)**
- **Default**: function comments should document **logic/intent only** via `/** Logic: ... */` or `// Logic: ...`.
- **Full JSDoc**: use `@param` / `@returns` only for shared/public APIs where the parameter or return contract is non-obvious and the docs add value.
- **Do not** add full `@param` / `@returns` blocks to short helpers, event handlers, page-level render/load functions, or straightforward functions just because they are functions.
- **Inline comments**: use `//` to mark major steps inside function body:
  ```js
  // 1. Validate input
  // 2. Parse/transform data
  // 3. Handle edge cases
  // 4. Return result
  ```
- Complex logic (loops, conditions, DOM manipulation): add explanatory comment above the block
- Avoid obvious comments (`// increment i` for `i++`); focus on **why**, not **what**

**When to SKIP the full JSDoc block (anti-bloat rule)**

Full `@param` / `@returns` / `Logic:` JSDoc is for **non-trivial shared/public API contracts only**. For ordinary page functions, handlers, helpers, callbacks, and one-liners it inflates the file far more than it documents — a 6-line `setCellValue` or `renderXxx` does not need a 9-line JSDoc above it.

Skip the full JSDoc block (use a single `// short description` line instead) when **all** of these are true:
- The function body is **≤ ~8 lines** and is a single conceptual step (no branching beyond an early-return / guard).
- It is an **inline callback** or option-bag handler — `onClick`, `onValueChanged`, `setCellValue`, `onInitNewRow`, comparator/predicate in `.sort()` / `.find()` / `.filter()`, an event handler passed to `.on(...)`, etc.
- The parameter shape is **already obvious from context** — e.g. the DevExtreme event object passed to `onValueChanged`, a row object in `setCellValue`, the standard `(rowA, rowB)` of a comparator.
- It is **not exported** and not reused from another file.

When skipping, still:
- Put a **one-line `//` comment** above the function explaining the intent (the "why", not the "what").
- Keep numbered inline step comments **only if** the body actually has multiple distinct steps worth labelling — don't fake-split a 4-line body into `// 1.` / `// 2.` / `// 3.` just to satisfy the checklist.
- Keep variable-naming rules (no single letters, `event` not `e`, etc.) — those apply regardless of size.

Write the full JSDoc block only when the function is a **shared/public API contract** and at least one of these is true:
- The parameter shape is **non-obvious** (custom object, multiple options, optional flags).
- The return shape is **non-obvious** and callers depend on it.
- The function is consumed across files/modules and the contract is not clear from the name/call site.

For page-local `renderXxx`, `loadXxx`, `onXxx`, route helpers, and straightforward business helpers, prefer `/** Logic: ... */` without `@param` / `@returns`.

Rule of thumb: if the JSDoc block is longer than the function body, delete the JSDoc and write a one-line comment.

**Validation & Edge Cases**
- Check `null`/`undefined`/empty string before processing
- Normalize input: `param = param || '';` or `param?.trim() === ''`
- Return consistent shape (e.g., `{ success: bool, data: any, error?: string }` if applicable)

**DOM Manipulation**
- Use safe APIs: `DOMParser`, `textContent`, avoid `innerHTML` with user input (XSS risk)
- Comment non-obvious DOM logic
- **jQuery-first**: In pages that already load jQuery, prefer jQuery APIs over vanilla DOM:
  - `$('#id')` not `document.getElementById('id')`
  - `$el.remove()` not `el?.remove()`
  - `$el.val()` not `el.value`
  - `$el.on('click', handler)` not `el.addEventListener('click', handler)`
  - `$el.focus()` not `el.focus()`
  - `$el.addClass()` / `$el.toggleClass()` not direct `.className =` assignment
  - Exception: performance-critical tight loops or code that must run before jQuery loads

**Readability**
- Use `const`/`let` (never `var` in modern code — exception: legacy or specific scope needs)
- Descriptive variable names (`resultBefore`, `resultAfter` not `a`, `b`)
- Split only complex or reused logic into helper functions and use concise logic comments by default

**Naming — no single-letter variables**
- **BANNED** as regular variables: `D`, `d`, `i`, `f`, `t`, `s`, `r`, `a`, `b`, `c`, `e`, `n`, `v`, `x`, `y`, `q`, `o` — even as a short alias for a store like `D = window.Store`
- Use the full, domain meaningful name: `store`, `detail`, `header`, `file`, `filters`, `task`, `row`, `response`, `activity`, `error`, `event`, `index`, `value`, `query`, `option`
- Loop / callback parameters: `forEach(item => …)`, `.each(function(index, row) {…})`, comparator `(rowA, rowB) => …`, not `.each(function(i, d) {…})`
- **Allowed single-letter exceptions** (only these):
  - Math axis coordinates: `x`, `y`, `w`, `h`
  - Regex match group: `m`
  - Arrow-function throw-away in inner reducers that never leave one line: `arr.reduce((acc, n) => acc + n, 0)` — prefer `acc + num` anyway
- When you find yourself typing a one-letter name, stop and rename it before moving on. Someone (often you, 6 months later) has to read this.

**Function extraction — when to split, when to inline**

Extract a function only when **one** of these is true:
- **Complex logic**: multiple branches, nested loops, or > ~15 lines that does a single conceptual thing — extracting gives the concept a name and lets readers skim the parent without diving in.
- **Reused**: called from 2+ sites — extract once, reference everywhere (DRY).

Do **NOT** extract when:
- The logic is short **and** called from a single place. A 3-line `if/return` inlined is easier to follow than a named helper that hides 3 lines.
- The "function" just renames a one-liner without adding clarity (`function getId(item) { return item.id; }` — pointless).
- Extracting forces the reader to jump around the file to follow a straight-line flow that would otherwise read top-to-bottom.

**Default: inline first.** Extract later only when reuse actually appears or the parent grows genuinely hard to read. Three similar inlined lines are better than three calls to a tiny helper — premature extraction hurts readability and makes the file harder to navigate, not easier.

Once you decide to extract, apply the **placement rule** (also stated in File Organization below):
- Called in **one** place → put it immediately below its parent.
- Called in **more than one** place → lift it to the HELPERS section.

**Cached DOM selectors (page-level jQuery code)**
- Declare module-level `$selector` variables once at the top (next to state) and assign them in a `cacheSelectors()` helper called from init
- Re-render methods must use the cached selector (`$headerComp.html(…)`), not `$('#headerComp').html(…)` — repeated `$('#…')` lookups are both slower and noisier to read
- Name caches after their DOM id / role: `$headerComp`, `$detailComp`, `$activityComp`, `$modalContainer`

**File Organization (page-level JS)**
Ordered top-to-bottom sections separated by `// ─── SECTION NAME ──` banners (em dashes, **not** `// ═══`):

1. **CONSTANTS** — `const` lookup maps, enums, fixed values (e.g. `ISSUE_TABS`, `STATUS_CLASS_MAP`, `TD_LEVELS`)
2. **STATE** — `let` module-level state (e.g. `issueHeader`, `issueDetail`, `activities`, `activityFilters`)
3. **CACHED SELECTORS** — `let $headerComp, $detailComp, …;` (assigned later in `cacheSelectors()`)
4. **HELPERS** — small pure utilities used by more than one section (e.g. `escapeHtml`, `formatDate`, `pillHtml`, `valueHtml`)
5. **URL / ROUTING** — URL parsing, history pushState, popstate handler
6. **API** — async data loaders + DTO → state mappers (e.g. `loadIssueData`, `mapIssueInfoToState`)
7. **Feature sections** — one banner per panel / component (`// ─── HEADER ──`, `// ─── DETAIL ──`, `// ─── ACTIVITY ──`). Inside each section put:
   - The main `renderXxx()` first
   - Sub-renderers and row-builders immediately below their parent
   - Event handlers (`onXxxClick`, `onXxxChange`) last in the section
8. **INIT HELPERS** — `cacheSelectors()`, `renderAll()`, `bindEvents()`
9. **INIT** — single `$(async function () { … })` at the very bottom that calls the init helpers in order

**Banner format example:**
```js
// ─── CONSTANTS ───────────────────────────────────────────────────
const VALID_TYPES = ['sprint', 'requirement', 'release', 'feature'];

// ─── STATE ──────────────────────────────────────────────────────
let selectedType = 'sprint';

// ─── CACHED SELECTORS ───────────────────────────────────────────
let $sidebar, $timelineWrap, $ganttHeader;
```

Sub-function placement rule: if a helper is called in **one** place → put it immediately below its parent. If it's called in **more than one** place → lift it to the HELPERS section.

Prefer flat functions over object-based component pseudo-classes. Objects scatter handlers into method references and make event binding harder to scan — flat `renderHeader()` / `onHeaderTabClick()` reads top-to-bottom.

---

## JS Template

```js
/**
 * Logic:
 * - If htmlBefore empty → entire htmlAfter = added (class "pass")
 * - If htmlAfter empty → entire htmlBefore = removed (class "fail")
 * - Otherwise → use Diff.diffWords() to wrap changes in <span>
 */
function diffHtml(htmlBefore, htmlAfter) {
    // 1. Normalize input
    htmlBefore = htmlBefore || '';
    htmlAfter = htmlAfter || '';

    // 2. Handle edge cases
    if (htmlBefore.trim() === '') {
        return { before: '', after: htmlAfter ? `<span class="pass">${htmlAfter}</span>` : '' };
    }
    if (htmlAfter.trim() === '') {
        return { before: `<span class="fail">${htmlBefore}</span>`, after: '' };
    }

    // 3. Parse HTML and extract text
    const textBefore = new DOMParser().parseFromString(htmlBefore, 'text/html').body.textContent || '';
    const textAfter = new DOMParser().parseFromString(htmlAfter, 'text/html').body.textContent || '';

    // 4. Compute word-level diff and build results
    const diff = Diff.diffWords(textBefore, textAfter);
    let resultBefore = '', resultAfter = '';

    diff.forEach(part => {
        if (!part.added && !part.removed) {
            resultBefore += part.value;
            resultAfter += part.value;
        } else if (part.added) {
            resultAfter += `<span class="pass">${part.value}</span>`;
        } else if (part.removed) {
            resultBefore += `<span class="fail">${part.value}</span>`;
        }
    });

    return { before: resultBefore, after: resultAfter };
}
```

---

## Checklist (AI/Developer)

- [ ] Function comments default to `/** Logic: ... */` or `// Logic: ...`; full `@param` / `@returns` JSDoc is only for non-obvious shared/public API contracts. **Anti-bloat rule: if the JSDoc is longer than the function body, drop it.**
- [ ] **Inline comments** marking major steps (// 1. Validate, // 2. Parse, etc.) — only when the body has multiple distinct steps worth labelling, not as filler for 3-line functions
- [ ] Input validation (null/empty checks)
- [ ] Safe DOM handling (DOMParser, textContent)
- [ ] **jQuery-first** — `$('#id')` not `document.getElementById()`, `$el.remove()` not `el?.remove()`, etc.
- [ ] `const`/`let` (not `var` unless legacy)
- [ ] Descriptive names + helper functions if complex
- [ ] **No single-letter variables** — `D`/`d`/`i`/`f`/`t` etc. renamed to domain terms (`store`, `detail`, `index`, `file`, `task`)
- [ ] **Cached `$selector` vars** declared at the top and assigned in `cacheSelectors()`; no repeated `$('#id')` lookups inside render methods
- [ ] Section banners use `// ─── SECTION NAME ──` (em dashes), not `// ═══ ═══`
- [ ] `$(async function () { … })` entry point lives at the very bottom and calls `cacheSelectors()` → `bindEvents()` → data load → render
- [ ] Flat `renderXxx` / `onXxxClick` functions grouped by feature banner, **not** object-based `HeaderComp.render()` pseudo-classes

---

## Prompt Examples

- **JS**: _"Create `validateForm(formData)` with concise `Logic:` comments: check required fields, return { valid: bool, errors: string[] }, handle null input."_

---

**End of frontend prompt file**

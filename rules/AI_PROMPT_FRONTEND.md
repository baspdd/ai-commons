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
- Comment **every** logic block; scale the comment to the unit's size — small helper → one-line `// Logic: ...`, medium function → `/** Logic: ... */` block, large/public/non-obvious function → full `@param` / `@returns` JSDoc.
- Structure files with `// ═══ SECTION ═══` banners — one per section by default. Add `// ─── sub-section ──` (em-dash) banners only inside a large section with several distinct sub-blocks (e.g. a feature panel's render / events), so a reader follows banner → function → inline steps top-down.
- Prefer early returns to reduce deep nesting.

## JavaScript/TypeScript Rules

**Function Structure — comment depth scales with function size**
- Every function carries a comment whose depth matches its size. **Medium** functions (branching / several steps) use a `/** Logic: ... */` block like:
  ```js
  /**
   * Logic:
   * - Key behavior or reason this function exists
   * - Important edge case only if it is not obvious from code
   */
  ```
- **Small** functions (≤ ~8 lines, single step) use a one-line `// Logic: ...` instead of the block — but still always get a comment.
- **Large / public / non-obvious-contract** functions use full JSDoc with `@param` / `@returns` (the bigger and more reused, the more it documents):
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
- **Every logic block is commented** — banners, function headers, and non-trivial blocks inside a body. A reader skimming major banner → sub banner → function header → inline steps should follow the logic top-down without reading the implementation.
- **Scale to size**: small helper → one-line `// Logic: ...`; medium function → `/** Logic: ... */`; large/public/non-obvious-contract function → full `@param` / `@returns` JSDoc. The bigger and more reused the function, the more it documents.
- **Inline comments**: use `//` to mark major steps inside function body:
  ```js
  // 1. Validate input
  // 2. Parse/transform data
  // 3. Handle edge cases
  // 4. Return result
  ```
- Complex logic (loops, conditions, DOM manipulation): add explanatory comment above the block
- Avoid obvious comments (`// increment i` for `i++`); focus on **why**, not **what**

**Comment depth scales with function size (every function commented)**

Match the comment to the unit's size — the goal is that **every logic block is documented**, not that comments are minimized. Together with the section banners, a reader follows section banner → (sub-section banner, where present) → function header → inline step comments top-down without reading the implementation.

- **Small** (≤ ~8 lines, single conceptual step; inline callbacks / option-bag handlers / `.sort()`·`.find()`·`.filter()` predicates included): a single `// Logic: ...` line above it. Always present — "too small to comment" is not an exemption; even a tiny helper gets a moderate one-line comment.
- **Medium** (branching, several steps, a real helper): a `/** Logic: ... */` block listing the key steps and non-obvious edge cases; add numbered `// 1. … // 2. …` inline comments for the distinct phases inside the body.
- **Large / public / cross-file / non-obvious contract**: full JSDoc with `@param` / `@returns` describing the parameter and return shape, plus a `Logic:` bullet list. The bigger and more reused the function, the more it must document.

Still applies at every size:
- Focus on **why**, not **what** — don't restate the code (`// increment i` for `i++`).
- Don't fake-split a short body into `// 1. / // 2.` just to fill the checklist — only label genuinely distinct phases.
- Naming rules hold regardless of size (no single letters, `event` not `e`, etc.).

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
Ordered top-to-bottom sections. **Scale banner depth to size:** by default give each section a single `// ═══ SECTION ═══` banner. Add a second `// ─── sub-section ──` (em-dash) level **only inside a large section that has several distinct sub-blocks** — typically a feature panel with render / row-builders / events, and especially when those sub-labels repeat across multiple panels (the `═══` banner is then the only thing telling you which panel you're in). Don't put a major banner over a single small section — it just duplicates the sub-label below it.

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

**Banner format — default: one `═══` banner per section, single level:**
```js
// ═══════════════════════════════════════════════════════════
// CONSTANTS
// ═══════════════════════════════════════════════════════════
const VALID_TYPES = ['sprint', 'requirement', 'release', 'feature'];

// ═══════════════════════════════════════════════════════════
// STATE
// ═══════════════════════════════════════════════════════════
let selectedType = 'sprint';

// ═══════════════════════════════════════════════════════════
// CACHED SELECTORS
// ═══════════════════════════════════════════════════════════
let $sidebar, $timelineWrap, $ganttHeader;
```

**Add a second `───` level only inside a large section whose sub-blocks repeat** — e.g. multiple feature panels that each have render / events, where the `═══` banner is what disambiguates the otherwise-identical sub-labels:
```js
// ═══════════════════════════════════════════════════════════
// HEADER
// ═══════════════════════════════════════════════════════════

// ─── RENDER ──────────────────────────────────────────────
function renderHeader() { … }

// ─── EVENTS ──────────────────────────────────────────────
function onHeaderTabClick(e) { … }

// ═══════════════════════════════════════════════════════════
// DETAIL
// ═══════════════════════════════════════════════════════════

// ─── RENDER ──────────────────────────────────────────────
function renderDetail() { … }

// ─── EVENTS ──────────────────────────────────────────────
function onDetailSave(e) { … }
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

- [ ] Every function commented, depth scaled to size — small: one-line `// Logic: ...`; medium: `/** Logic: ... */`; large/public/non-obvious: full `@param` / `@returns` JSDoc.
- [ ] **Inline comments** marking major steps (// 1. Validate, // 2. Parse, etc.) — only when the body has multiple distinct steps worth labelling, not as filler for 3-line functions
- [ ] Input validation (null/empty checks)
- [ ] Safe DOM handling (DOMParser, textContent)
- [ ] **jQuery-first** — `$('#id')` not `document.getElementById()`, `$el.remove()` not `el?.remove()`, etc.
- [ ] `const`/`let` (not `var` unless legacy)
- [ ] Descriptive names + helper functions if complex
- [ ] **No single-letter variables** — `D`/`d`/`i`/`f`/`t` etc. renamed to domain terms (`store`, `detail`, `index`, `file`, `task`)
- [ ] **Cached `$selector` vars** declared at the top and assigned in `cacheSelectors()`; no repeated `$('#id')` lookups inside render methods
- [ ] Banner depth scaled to size — one `// ═══ SECTION ═══` per section by default; add `// ─── sub-section ──` (em-dash) only inside a large section whose sub-blocks repeat across panels (no major banner over a single small section)
- [ ] `$(async function () { … })` entry point lives at the very bottom and calls `cacheSelectors()` → `bindEvents()` → data load → render
- [ ] Flat `renderXxx` / `onXxxClick` functions grouped by feature banner, **not** object-based `HeaderComp.render()` pseudo-classes

---

## Prompt Examples

- **JS**: _"Create `validateForm(formData)` with concise `Logic:` comments: check required fields, return { valid: bool, errors: string[] }, handle null input."_

---

**End of frontend prompt file**

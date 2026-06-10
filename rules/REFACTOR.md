# Refactoring Rules (Language-Agnostic)

> **Usage:** Combine this file with a language-specific rule file:
> - Frontend JS/TS → `REFACTOR.md` + `AI_PROMPT_FRONTEND.md`
> - Backend C#/ABP → `REFACTOR.md` + `AI_PROMPT_BACKEND.md`
> - SQL PostgreSQL → `REFACTOR.md` + `AI_PROMPT_SQL.md`

Read the uploaded file and refactor it according to **both** this file and the matching language rule file.

---

## General Principles

- **Preserve all original behavior** — no logic changes, no feature additions
- **Do NOT remove any comments** (inline, block, JSDoc, XML doc, `#region`)
- Keep the same variable and function names unless renaming improves clarity or removes a banned name (see language rule file)
- Compact redundant or repetitive code where possible
- Merge similar logic if safe to do so
- Remove unnecessary whitespace or blank lines (keep one blank line between logical groups)

---

## Code Organization

- **Reorganize into logical sections** separated by clear banners / regions (format defined in language rule file)
- Group related functions together by feature or responsibility
- Sub-function placement:
  - Called in only **ONE** place → place directly below its caller
  - Called in **MORE than one** place → lift to a shared helpers/utilities section

---

## Readability & Naming

- Use descriptive, domain-meaningful variable names — no ambiguous abbreviations
- Banned single-letter variable names are defined in each language rule file — rename all violations
- Split complex logic into well-named helper functions
- Constants / magic values → extract to named constants at the top of the file

---

## Comments & Documentation

- Function comments must follow the matching language rule file; JS/TS defaults to concise `Logic:` comments and uses full JSDoc only when the frontend rule requires it
- Add inline step comments (`// 1.`, `// 2.`, …) to any function body with more than one logical phase
- Focus on **why**, not **what** — avoid obvious comments (`// increment i`, `// return result`)
- Complex logic (loops, conditions, transformations): add an explanatory comment above the block

---

## Error Handling & Validation

- Check `null` / `undefined` / empty / invalid values before processing
- Follow the error-handling pattern from the language rule file (try/catch, early return, response shape)
- Don't add error handling for scenarios that can't happen — only validate at system boundaries

---

## Output

Return the **full refactored file content**, ready to copy-paste.
If a section doesn't exist in the original file, skip that section header silently.

---

## Checklist (applies to all languages)

- [ ] Behavior preserved — no logic changes
- [ ] No removed comments
- [ ] Code organized into logical sections with banners/regions
- [ ] Descriptive variable names — no banned single-letter names
- [ ] Function comments follow the matching language rule file; JS/TS uses concise `Logic:` comments by default
- [ ] Inline step comments (`// 1.`, `// 2.`) on multi-step functions
- [ ] Redundant / repetitive code compacted
- [ ] Constants extracted — no magic values
- [ ] Error handling follows language-specific pattern
- [ ] See language-specific checklist for additional items

---

## Git Commit Standards

**Format:** `[type] author | Description`

**Types:** `feat` / `upd` / `fix` / `refactor` / `docs` / `test`

**Examples:**
- `[feat] duy7974 | Add requirement level filtering`
- `[fix] duy7974 | Fix null reference in requirement phase calculation`

**Rules:** Use present tense, keep description concise, group related changes

---
name: load-rules
description: Use before the first edit to C#, frontend, SQL, permission, localization, or refactor scope when canonical rules have not been loaded.
---

# load-rules

This skill makes sure the matching rule files are loaded into the active session before you write or modify code. Loading is cheap; rule-violating code is expensive.

> **Path note**: the canonical rule folder is `rules`.

## When to invoke

- User asks to edit / create / refactor a `.cs`, `.js`, `.ts`, `.cshtml`, or `.sql` file **and** the rules for that category have not yet been loaded in this session
- Beginning of a coding task where the file type is known
- Switching to a new file type mid-session (e.g. backend → SQL migration)

Skip when the task is purely conversational (questions, explanations, review of unchanged files).

## What to load

Detect the dominant file type from the request, then `Read` the matching files:

| File category | Read this file |
|---|---|
| `*.cs` (C# / ABP / EF Core) | `rules/AI_PROMPT_BACKEND.md` |
| `*.js` / `*.ts` / page-level JS / `*.cshtml` JS blocks | `rules/AI_PROMPT_FRONTEND.md` |
| `*.sql` / `Source/database/V*.sql` / Postgres functions | `rules/AI_PROMPT_SQL.md` |
| `Permissions/*.cs` (constants / provider) or `Menus/*MenuContributor.cs` | `rules/MENU_PERMISSION.md` **plus** the matching language file above |
| Localization JSON files under `**/Localization/**/*.json` (en/vi/jp) | `rules/LOCALIZATION.md` |
| Any **refactor** task (rewrite / restructure / extract / rename) | `rules/REFACTOR.md` **plus** the matching language file above |

The host project's `.ai/PROJECT.md` is always relevant. Load it on the **first**
rule-loading call of a session, then skip it on later calls.

## How

1. Identify file type from the user request and any file paths mentioned.
2. If the task spans multiple types (e.g. AppService + Razor page + SQL migration), load each category the first time you touch it.
3. Read each required file with the `Read` tool. Don't re-read a file already loaded in this session.
4. If unsure whether a rule file has been loaded, **re-read** it — that's cheaper than producing code that violates it.
5. After loading, give one short sentence stating which rule sets are now active, then proceed with the user's task.

## Output

Reply with a single line per loaded file:
```
Loaded: <path>
```

Then a closing line:
```
Ready. Active rules: <Backend | Frontend | SQL | Refactor | ...>
```

Nothing else — keep this skill minimal so the real work follows immediately.

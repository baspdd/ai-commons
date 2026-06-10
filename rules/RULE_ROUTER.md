# Shared AI Rule Router

This file is the canonical routing table for Claude, Codex, and other coding agents.
Load detailed rules only when the task touches their scope.

| Scope | Canonical rule |
|---|---|
| C#, ABP, EF Core, Application Services, Domain, repositories | `rules/AI_PROMPT_BACKEND.md` |
| JavaScript, TypeScript, Razor scripts, jQuery, DevExtreme | `rules/AI_PROMPT_FRONTEND.md` |
| PostgreSQL, raw SQL, versioned migrations | `rules/AI_PROMPT_SQL.md` |
| Permissions or menu contributors | `rules/MENU_PERMISSION.md` plus the language rule |
| Localization JSON | `rules/LOCALIZATION.md` |
| Refactoring | `rules/REFACTOR.md` plus the language rule |
| Host project architecture | the project's `.ai/PROJECT.md` |

## Loading Policy

1. Read the actual code and determine which scopes are affected.
2. Load each matching rule before the first edit in that scope.
3. Load only relevant rules; do not copy their checklists into global configuration.
4. After editing, review the change against the same canonical rules and fix violations
   before reporting completion.
5. Preserve project-local instructions when they are stricter than this shared base.

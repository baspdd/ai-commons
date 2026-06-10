---
name: new-migration
description: Use when adding a versioned PostgreSQL migration for a table, column, function, index, or data correction.
---

# new-migration

Creates a new versioned PostgreSQL migration file following `AI_PROMPT_SQL.md`.

## When to invoke

- User asks to "create a migration / migration file / new table / drop column / add function"
- User says "viết file SQL để thêm bảng / cột / function"
- Schema change that the EF Core migration generator can't (or shouldn't) handle alone — Postgres functions, complex indices, data backfills

Do NOT invoke when:
- A pure EF Core migration is sufficient (no raw SQL) — the user should run `dotnet ef migrations add` instead
- The user is editing an existing migration file — use `compliance-check` afterward

## Inputs to gather (ask if missing)

1. **Migration purpose** (one-line: e.g. "add WM_RequirementPhaseStatus lookup table", "add GetIssueScoresByMilestone function")
2. **Ticket / issue number** (for the header)
3. **Migration type** — `table` / `function` / `column-add` / `column-drop` / `index` / `data-fix`
4. **Target table or function name** (PascalCase, `WM_` prefix for tables)

## Procedure

1. **Load rules** (once per session):
   - `rules/AI_PROMPT_SQL.md`

2. **Find next version number.** `Glob` `Source/database/V*.sql`, sort lexicographically, pick the next available version per the project's pattern (confirm against the most recent file's naming).

3. **Compose filename** following the existing convention seen in the folder. Typical pattern: `V<num>__<snake_case_description>.sql` — but **match what's actually used in the repo** (check 2–3 recent files first).

4. **Generate header** at the top of the file:
   ```sql
   -- =============================================================================
   -- Version    : V<num>
   -- Date       : <YYYY-MM-DD>
   -- Author     : <git user.name>
   -- Ticket     : <ticket>
   -- Summary    : <one line>
   -- =============================================================================
   ```

5. **Emit the body** based on migration type. Always idempotent. Templates:

   **a. New entity table**

   Read the C# entity first. Add audit columns according to its base class/interfaces.
   The following full set is only for a full-audited aggregate/entity that also implements
   extra-properties and concurrency-stamp contracts:
   ```sql
   DROP TABLE IF EXISTS public."WM_<Name>";

   CREATE TABLE IF NOT EXISTS public."WM_<Name>"
   (
       "Id" uuid PRIMARY KEY,
       -- domain columns ...
       -- Include only fields required by the actual entity contract.
       "ExtraProperties" text COLLATE pg_catalog."default" DEFAULT '{}',
       "ConcurrencyStamp" character varying(40) COLLATE pg_catalog."default",
       "CreationTime" timestamp without time zone NOT NULL,
       "CreatorId" uuid,
       "LastModificationTime" timestamp without time zone,
       "LastModifierId" uuid,
       "IsDeleted" boolean NOT NULL DEFAULT false,
       "DeleterId" uuid,
       "DeletionTime" timestamp without time zone
   )
   TABLESPACE pg_default;

   ALTER TABLE IF EXISTS public."WM_<Name>" OWNER TO dev_wm;
   ```

   **b. New function**
   ```sql
   DROP FUNCTION IF EXISTS "<FunctionName>"(<param_types>);

   CREATE OR REPLACE FUNCTION "<FunctionName>"(<param_name> <param_type>)
   RETURNS TABLE(...) -- or RETURNS <type>
   LANGUAGE sql       -- prefer over plpgsql when no procedural logic needed
   STABLE             -- VOLATILE if mutating or non-deterministic
   AS $$
       -- query
   $$;

   ALTER FUNCTION "<FunctionName>"(<param_types>) OWNER TO dev_wm;
   ```

   **c. Add column (additive, idempotent)**
   ```sql
   ALTER TABLE public."WM_<Name>"
       ADD COLUMN IF NOT EXISTS "<ColumnName>" <type> <constraints>;
   ```

   **d. Drop column / table** — pair with a clear comment explaining why; idempotent `DROP ... IF EXISTS`.

   **e. Data fix** — wrap in `BEGIN ... COMMIT`, include a `SELECT count(*)` sanity check comment showing how to verify before running.

6. **Self-review** — run the `compliance-check` skill (SQL section). Fix every finding.

## Output

```
## Created migration

File: Source/database/V<num>__<description>.sql

Highlights:
- <one-line: what changes>
- <one-line: idempotency / rollback note>

Next steps (not done by this skill):
- Test on local dev DB: psql -U dev_wm -d <dbname> -f Source/database/V<num>__<description>.sql
- Run sql-review CI stage to confirm
- Commit with message: [feat] <author> | <short desc>
```

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

3. **Compose filename** — `V{YYYYMMDD}_{seq}__{description}.sql`, matching the description style
   already used in the folder (Work Management: `SQL_v{semver}`). Check the two most recent files.

4. **Generate header** at the top of the file — all five fields, ISO date, banner 93 `=`. This is
   fixed; do not shorten it, do not copy a different shape from an older file:
   ```sql
   -- =============================================================================================
   -- Version    : V<num>
   -- Date       : <YYYY-MM-DD>
   -- Author     : <git user.name>
   -- Ticket     : <ticket, or "-" when there genuinely is none>
   -- Summary    : <one imperative line>
   -- =============================================================================================
   ```
   Rationale, measurements and deployment notes belong below the banner under their own sub-banner
   headings — not inside `Summary`.

   **Every comment in the file is English**, including `RAISE EXCEPTION` / `RAISE NOTICE` messages.

5. **Open the transaction** and declare the session settings before any section:
   ```sql
   BEGIN;

   SET LOCAL TIME ZONE 'UTC';
   SET LOCAL datestyle = 'ISO, YMD';
   SET LOCAL lock_timeout = '60s';
   ```
   Take `LOCK TABLE public."WM_<Name>" IN SHARE ROW EXCLUSIVE MODE;` before any read-then-write step
   that concurrent writers could corrupt. Close the file with `COMMIT;`.

6. **Emit the body** as numbered sections, each under a sub-banner (93 `-`):
   ```sql
   -- ---------------------------------------------------------------------------------------------
   -- 1. <what this section does, and why>
   -- ---------------------------------------------------------------------------------------------
   ```
   Always idempotent. Templates:

   **a. New entity table**

   Read the C# entity first. Add audit columns according to its base class/interfaces.
   The following full set is only for a full-audited aggregate/entity that also implements
   extra-properties and concurrency-stamp contracts:
   ```sql
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

   **b. New or changed function** — `CREATE OR REPLACE` only, never a `DROP`. If the signature has
   to change, create a **new function under a new name** and repoint the callers; the obsolete one
   is dropped by hand after the deploy, outside this file.
   ```sql
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

   **d. Retiring a column / table / function** — the migration does **not** drop it. Stop writing to
   it in code, note the retirement in a section comment, and hand the operator the `DROP` statement
   to run manually once the deploy is confirmed. A drop inside a migration is unrecoverable and
   cascades.

   **e. Data fix** — the `UPDATE` predicate must match nothing on a second run. Include a
   `SELECT count(*)` sanity-check comment showing how to verify before and after.

7. **Lint** — run `sqlfluff lint Source/database/V<num>__<description>.sql` against the
   project `.sqlfluff` (dialect=postgres). Fix every violation (`sqlfluff fix` for mechanical
   ones); do not reformat against the config's excluded rules.

8. **Self-review** — run the `compliance-check` skill (SQL section). Fix every finding.

## Output

```
## Created migration

File: Source/database/V<num>__<description>.sql

Highlights:
- <one-line: what changes>
- <one-line: idempotency / rollback note>

Next steps (not done by this skill):
- Lint: sqlfluff lint Source/database/V<num>__<description>.sql   # project .sqlfluff, dialect=postgres
- Test on local dev DB: psql -U dev_wm -d <dbname> -f Source/database/V<num>__<description>.sql
- Run sql-review CI stage to confirm
- Commit with message: [feat] <author> | <short desc>
```

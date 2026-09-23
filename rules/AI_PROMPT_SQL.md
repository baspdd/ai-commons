# AI Prompt: SQL (PostgreSQL) Rules

> **For refactoring:** Combine with `REFACTOR.md` which contains the general refactoring process.

## Must-Follow Rules

**Naming Conventions**
- Tables: `"WM_PascalCase"` — prefix `WM_`, double-quoted, PascalCase (e.g. `"WM_ProjectNodes"`, `"WM_RequirementPhases"`)
- Columns: `"PascalCase"` — double-quoted (e.g. `"ProjectId"`, `"CreationTime"`)
- Functions: `"PascalCase"` — case-sensitive, mirror .NET method names (e.g. `"BatchIssueScoresAsync"`, `"GetBugDetail"`)
- Local variables (plpgsql): `snake_case` — no quotes (e.g. `from_index`, `prev_tc_id`)
- CTE names: `snake_case` — descriptive (e.g. `issue_type_weights`, `user_role_weights`)
- Parameters: `snake_case` prefixed with `_` or `p_` (e.g. `_testrunid`, `p_testcase_ids`)

**ABP Audit Columns (derive from entity contract)**
Do not add a fixed nine-column audit set to every entity table. Read the C# entity base
class and implemented ABP interfaces, then persist only the fields required by that
contract:

- `IHasCreationTime` / `IMayHaveCreator` → `CreationTime`, optional `CreatorId`
- `IHasModificationTime` / `IModificationAuditedObject` → `LastModificationTime`,
  optional `LastModifierId`
- `ISoftDelete` → `IsDeleted`
- `IDeletionAuditedObject` → `DeletionTime`, optional `DeleterId`
- `IHasExtraProperties` → `ExtraProperties`
- `IHasConcurrencyStamp` → `ConcurrencyStamp`

A `FullAuditedEntity<TKey>` or full-audited aggregate root normally requires:
```sql
"ExtraProperties" text COLLATE pg_catalog."default" DEFAULT '{}',
"ConcurrencyStamp" character varying(40) COLLATE pg_catalog."default",
"CreationTime" timestamp without time zone NOT NULL,
"CreatorId" uuid,
"LastModificationTime" timestamp without time zone,
"LastModifierId" uuid,
"IsDeleted" boolean NOT NULL DEFAULT false,
"DeleterId" uuid,
"DeletionTime" timestamp without time zone
```

An entity using a narrower base class must use its narrower field set. Verify the entity,
EF mapping, and existing schema before generating a migration.

**Table Creation Pattern**
```sql
CREATE TABLE IF NOT EXISTS public."WM_TableName"
(
    "Id" uuid PRIMARY KEY,           -- uuid for entity tables
    "SubId" serial,                   -- auto-increment display ID (when needed)
    -- ... domain columns ...
    -- ... audit columns required by the entity base class/interfaces ...
)
TABLESPACE pg_default;

ALTER TABLE IF EXISTS public."WM_TableName"
    OWNER TO dev_wm;
```
- **No `DROP TABLE` in a migration either** — same reasoning as functions, and it destroys data.
  A table that has to be replaced gets a new name plus a copy step; the obsolete one is dropped by
  hand afterwards. `CREATE TABLE IF NOT EXISTS` already makes a re-run harmless.
- Primary key: `uuid` for entity tables, `serial` for config/lookup tables
- `"SubId" serial` for human-readable sequential IDs
- Always set `OWNER TO dev_wm`
- Always use `TABLESPACE pg_default`
- Text columns: `text COLLATE pg_catalog."default"`

**Function Creation Pattern**
```sql
CREATE OR REPLACE FUNCTION "FunctionName"(param_name param_type)
RETURNS TABLE(...) | RETURNS type
LANGUAGE sql | plpgsql
STABLE | VOLATILE
AS $$ | AS $BODY$
    ...
$$ | $BODY$;

ALTER FUNCTION "FunctionName"(param_types)
    OWNER TO dev_wm;
```
- **Never `DROP FUNCTION` in a migration — `CREATE OR REPLACE` only.** Between the `DROP` and the
  `CREATE` the function does not exist, so any concurrent call fails outright, and `DROP ... CASCADE`
  additionally takes the dependent triggers and views with it, silently.
- **A signature change means a new function under a new name.** `CREATE OR REPLACE` cannot change
  the return type or the parameter list — Postgres either rejects it or quietly creates a second
  overload. Write the new name, repoint every caller, leave the old function in place. Removing the
  obsolete one is a manual step run separately once the deploy is confirmed; it never ships inside
  the migration file.
- `LANGUAGE sql` for pure read-only queries (prefer over plpgsql when no procedural logic needed)
- `LANGUAGE plpgsql` for procedural logic (loops, variables, conditionals)
- `STABLE` for read-only functions, `VOLATILE` for functions that modify data or depend on mutable state
- Always set `OWNER TO dev_wm`

**Query Patterns**
- **Soft-delete filter**: Include `WHERE "IsDeleted" = false` when querying tables whose
  entity implements `ISoftDelete`, unless the query intentionally includes deleted rows
- **NULL handling**: Use `COALESCE(value, default)` — never leave NULLs unhandled in calculations
- **CTEs over subqueries**: Use `WITH ... AS (...)` for readability; chain multiple CTEs with comments
- **DISTINCT ON**: Use `DISTINCT ON (key) ... ORDER BY key, sort_col DESC` for "latest row per group"
- **Array parameters**: Use `= ANY(array_param)` not `IN (SELECT unnest(...))`
- **LATERAL joins**: Use for correlated subqueries that need outer row reference
- **UUID generation**: `uuid_generate_v4()`
- **Date formatting**: `TO_CHAR(timestamp, 'YYYY-MM-DD')`
- **Type casting**: Explicit casts `::float8`, `::text`, `::varchar(40)`, `::uuid`
- **Sequence reset**: After bulk insert with serial, reset via:
  ```sql
  SELECT setval(
      pg_get_serial_sequence('public."WM_TableName"', 'SubId'),
      COALESCE((SELECT MAX("SubId") FROM public."WM_TableName"), 1),
      true
  );
  ```

**CTE Style**
```sql
WITH first_cte AS (
    -- Explain what this CTE resolves
    SELECT ...
),
second_cte AS (
    -- Explain what this CTE resolves
    SELECT ...
)
-- Final query: explain the join/aggregation purpose
SELECT ...
FROM first_cte
LEFT JOIN second_cte ON ...;
```
- Comment above or inside each CTE explaining its purpose
- One CTE per logical step — don't cram multiple concerns into one

**Trigger Pattern**
```sql
CREATE OR REPLACE TRIGGER trigger_name
    AFTER INSERT OR DELETE OR UPDATE
    ON public."WM_TableName"
    FOR EACH ROW
    EXECUTE FUNCTION public.trigger_function();
```

**Security & Safety**
- Never concatenate user input into SQL strings — use parameterized queries
- No `DROP` of any kind inside a migration — `CREATE OR REPLACE` for functions, a new name for a
  signature change, `IF NOT EXISTS` for tables and columns. Removing the obsolete object is a
  manual step, run separately once the deploy is confirmed
- Wrap every migration in `BEGIN ... COMMIT` with the session settings below
- Test destructive operations (`DELETE`, `UPDATE` without WHERE) in staging first

**Soft Delete in Raw SQL**
The ABP soft-delete filter is an EF Core query filter. It applies to LINQ only — never to
`FromSqlRaw`, `ExecuteSqlRaw`, `NpgsqlCommand`, PostgreSQL functions, views, or matviews. Raw SQL
that omits the predicate returns deleted rows silently: no error, no warning, just wrong numbers.
- Every raw read of a soft-delete table must carry `"IsDeleted" = false` itself, once per alias.
  A single predicate does not cover a second join onto the same table.
- Never `DELETE FROM` a soft-delete table. Use
  `UPDATE ... SET "IsDeleted" = true, "DeletionTime" = now() AT TIME ZONE 'utc', "DeleterId" = ...`
  so the row keeps its audit trail and stays restorable.
- A unique index on a soft-delete table must be partial (`WHERE "IsDeleted" = false`). A plain
  unique index makes a deleted row block the re-creation of the same key forever.
- **Promoting a table from hard delete to soft delete is a breaking change to every raw query that
  already reads it.** Before shipping such a migration, audit the live catalog — not just the
  repository, because most functions in this product exist only on the database and appear in no
  `.sql` file:
  ```sql
  SELECT p.proname, pg_get_functiondef(p.oid)
  FROM pg_proc AS p
  JOIN pg_namespace AS n ON n.oid = p.pronamespace
  WHERE n.nspname = 'public'
    AND p.prokind IN ('f', 'p')          -- pg_get_functiondef() raises 42809 on aggregates
    AND pg_get_functiondef(p.oid) LIKE '%<TableName>%';
  ```
  Repeat on every tenant database: they drift, so a function absent from one may exist on another.
  `V20260921_01__SQL_v2.6.0.sql` section 8 shows this audit embedded in the migration itself.

**Migration File Conventions**

This layout is fixed. A migration that deviates is wrong even if it runs — the whole point is that
any file in `Source/database/` can be read the same way by the next person.

*File naming* — `V{YYYYMMDD}_{seq}__{description}.sql` (Flyway format, double underscore). The
description follows the host repository; Work Management uses `SQL_v{semver}`, e.g.
`V20260923_01__SQL_v2.6.0.sql`.

*Header* — mandatory, all five fields, never partial. Date is **ISO `YYYY-MM-DD`**, banner rule is
93 `=` characters:

```sql
-- =============================================================================================
-- Version    : V20260730_02
-- Date       : 2026-07-30
-- Author     : <git user.name>
-- Ticket     : 12770
-- Summary    : One imperative line saying what this migration changes.
-- =============================================================================================
```

- `Version` repeats the filename stem. `Ticket` is required — write `-` only when a change genuinely
  has no ticket.
- `Summary` is one line. Rationale, measurements and deployment notes go **below** the banner under
  their own sub-banner headings, never crammed into `Summary`.

*Comment language* — **English**, everywhere: header, section headings, inline comments,
`RAISE EXCEPTION` and `RAISE NOTICE` messages.

*Sections* — numbered, each introduced by a sub-banner (93 `-`), explaining why, not restating the
SQL:

```sql
-- ---------------------------------------------------------------------------------------------
-- 1. Audit columns for WM_RelateTables
-- ---------------------------------------------------------------------------------------------
```

Do not use bare `-- DD/MM/YYYY` date headers: the date already lives in the filename and the header
block.

*Section order within a migration:*
  1. Table creates
  2. ALTER TABLE (add / rename columns)
  3. Function creates (`CREATE OR REPLACE` only)
  4. Trigger creation
  5. Data seeding (INSERT / UPDATE)
  6. Sequence resets

*Transaction wrapper* — one transaction per file, with explicit session settings, so a failure
halfway through leaves nothing behind:

```sql
BEGIN;

SET LOCAL TIME ZONE 'UTC';
SET LOCAL datestyle = 'ISO, YMD';
SET LOCAL lock_timeout = '60s';

-- numbered sections ...

COMMIT;
```

- `TIME ZONE 'UTC'` / `datestyle`: the file must behave identically whatever the operator's session
  defaults are.
- `lock_timeout`: the migration fails fast instead of queueing behind a long-running read and
  blocking the whole application while it waits.
- Take an explicit lock before any read-then-write step whose correctness depends on writers being
  stopped:
  ```sql
  LOCK TABLE public."WM_TableName" IN SHARE ROW EXCLUSIVE MODE;
  ```
  `SHARE ROW EXCLUSIVE` blocks writers while still allowing read-only diagnostics.
  `ACCESS EXCLUSIVE` only for a table being rewritten or retired.
- Guard preconditions instead of assuming them — `RAISE EXCEPTION` inside a `DO $migration$` block
  when a required table, column or constraint is absent, so the transaction aborts with a readable
  message rather than a constraint violation 300 lines later.
- Every statement idempotent: `IF NOT EXISTS`, `ADD COLUMN IF NOT EXISTS`, and `UPDATE` predicates
  that match nothing on a second run.

Reference implementation: `V20260730_02__SQL_v2.3.0.sql`.

**SQL Formatting & Linting (sqlfluff)**
Every `.sql` file (migrations, functions, ad-hoc scripts) must pass `sqlfluff lint` cleanly
using the host project's root `.sqlfluff` config before commit. The config pins
`dialect = postgres` and `max_line_length = 120`.
- Run `sqlfluff lint <file>` and resolve every violation. `sqlfluff fix <file>` may apply
  mechanical fixes, but review the result against the naming/layout rules above.
- The project `.sqlfluff` deliberately excludes rules that clash with WM conventions —
  `CP01`, `CP04`, `CP05`, `LT01`, `LT02`, `LT03`, `LT05`, `LT08`, `LT12`, `LT15`, `RF06`.
  Do not re-enable them or reformat against them: quoted `"PascalCase"` identifiers and the
  existing layout take precedence.
- Keep every line within `max_line_length = 120`.

---

## Checklist (AI/Developer)

- [ ] Table/column names: `"WM_PascalCase"` / `"PascalCase"`, double-quoted
- [ ] Function names: case-sensitive `"PascalCase"`, mirror .NET method names
- [ ] No `DROP` of any kind in the migration — `CREATE OR REPLACE`; a signature change gets a new
      function name, and the obsolete object is dropped by hand after deploy
- [ ] `LANGUAGE sql` + `STABLE` for read-only; `plpgsql` + `VOLATILE` for procedural/mutating
- [ ] `OWNER TO dev_wm` on all tables and functions
- [ ] Audit columns match the entity's actual base class/interfaces
- [ ] `WHERE "IsDeleted" = false` on reads of soft-delete entities unless intentionally bypassed —
      once per alias, and in raw SQL too (the ABP filter does not reach it)
- [ ] No `DELETE FROM` on a soft-delete table; unique indexes on one are partial
- [ ] Turning a table into a soft-delete table: live `pg_proc` audit done on every tenant database
- [ ] `COALESCE` for all nullable values in calculations
- [ ] CTEs with descriptive `snake_case` names and comments
- [ ] No string concatenation for parameters — use `$1` / named params
- [ ] Sequence reset after bulk insert with serial columns
- [ ] Migration file follows `V{YYYYMMDD}_{seq}__{description}.sql` naming
- [ ] Header banner carries all five fields — Version / Date (ISO `YYYY-MM-DD`) / Author / Ticket /
      Summary
- [ ] Sections numbered `-- 1.` / `-- 2.` under a sub-banner; no bare `-- DD/MM/YYYY` headers
- [ ] Every comment and `RAISE` message in English
- [ ] Wrapped in `BEGIN ... COMMIT` with `SET LOCAL` time zone / datestyle / `lock_timeout`, and an
      explicit `LOCK TABLE` before any read-then-write step
- [ ] SQL passes `sqlfluff lint` with the project `.sqlfluff` (dialect=postgres); no new violations

---

**End of SQL prompt file**

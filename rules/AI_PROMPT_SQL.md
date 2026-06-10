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
DROP TABLE IF EXISTS public."WM_TableName";

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
- Primary key: `uuid` for entity tables, `serial` for config/lookup tables
- `"SubId" serial` for human-readable sequential IDs
- Always set `OWNER TO dev_wm`
- Always use `TABLESPACE pg_default`
- Text columns: `text COLLATE pg_catalog."default"`

**Function Creation Pattern**
```sql
-- Drop previous version first
DROP FUNCTION IF EXISTS "FunctionName"(param_types);

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
- `DROP FUNCTION IF EXISTS` before `CREATE OR REPLACE` to handle signature changes
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
- Always `DROP ... IF EXISTS` before `CREATE OR REPLACE` for functions (handles signature changes)
- Use `IF NOT EXISTS` for `CREATE TABLE`
- Wrap multi-step data modifications in transactions (plpgsql `BEGIN ... EXCEPTION ... END`)
- Test destructive operations (`DROP`, `DELETE`, `UPDATE` without WHERE) in staging first

**Migration File Conventions**
- File naming: `V{YYYYMMDD}_{seq}__{description}.sql` (Flyway format, double underscore)
- Date comment headers: `-- DD/MM/YYYY` at the top of each section
- Section order within a migration:
  1. Table drops / creates
  2. ALTER TABLE (add/rename/drop columns)
  3. Function drops / creates
  4. Trigger creation
  5. Data seeding (INSERT / UPDATE)
  6. Sequence resets

---

## Checklist (AI/Developer)

- [ ] Table/column names: `"WM_PascalCase"` / `"PascalCase"`, double-quoted
- [ ] Function names: case-sensitive `"PascalCase"`, mirror .NET method names
- [ ] `DROP FUNCTION IF EXISTS` before `CREATE OR REPLACE FUNCTION`
- [ ] `LANGUAGE sql` + `STABLE` for read-only; `plpgsql` + `VOLATILE` for procedural/mutating
- [ ] `OWNER TO dev_wm` on all tables and functions
- [ ] Audit columns match the entity's actual base class/interfaces
- [ ] `WHERE "IsDeleted" = false` on reads of soft-delete entities unless intentionally bypassed
- [ ] `COALESCE` for all nullable values in calculations
- [ ] CTEs with descriptive `snake_case` names and comments
- [ ] No string concatenation for parameters — use `$1` / named params
- [ ] Sequence reset after bulk insert with serial columns
- [ ] Migration file follows `V{YYYYMMDD}_{seq}__{description}.sql` naming
- [ ] Date headers (`-- DD/MM/YYYY`) separate logical sections

---

**End of SQL prompt file**

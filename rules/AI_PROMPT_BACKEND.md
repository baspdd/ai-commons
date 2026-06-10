# Backend Rules (C# / ABP / EF Core)

> For refactoring, also load `REFACTOR.md`.

## Core Principle

Backend flow follows:

```text
Application Service -> Domain logic -> Repository data access
```

Each layer owns a different responsibility. Do not move business rules into repositories
just to keep an AppService visually short.

## Layer Responsibilities

### Application Service

Application Services implement application use cases and expose the public contract.

- Authorize the use case with `[Authorize(...)]` or an equivalent scope check.
- Validate request shape and use-case preconditions.
- Load aggregates or query models through repositories.
- Call entity behavior, domain managers, or domain services for business decisions.
- Coordinate multiple domain operations, repositories, external services, history hooks,
  notifications, and the ABP Unit of Work.
- Map entities/query results to DTOs and preserve the existing public response contract.
- Keep persistence details out: no direct SQL and no reusable EF query construction.

An AppService may contain orchestration logic. It should not duplicate domain invariants or
contain low-level data-access logic.

### Domain

The Domain layer owns business meaning and invariants.

- Entities and aggregate roots protect valid state transitions.
- Value objects model domain concepts.
- Domain managers/services handle business rules that span entities or aggregates.
- Repository interfaces describe persistence capabilities needed by the domain/application.
- Domain code must not reference Application, HttpApi, Web, EF Core, or database-specific
  types.

Put a rule in Domain when it must remain true regardless of which endpoint, background job,
or integration invokes the operation.

### Repository

Repositories implement persistence and query access.

- Encapsulate EF Core, SQL, database joins, projections, batching, and persistence concerns.
- Return entities, typed DTOs/query models, scalars, or collections appropriate to the
  repository contract.
- Do not return HTTP/API envelopes such as `{ isSuccess, data, msg }`.
- Do not decide business outcomes, permissions, workflow transitions, or user-facing
  messages.
- Do not catch every exception merely to convert it into an API response. Preserve database
  exception context for the application/framework boundary.

Repository interfaces belong in Domain when they operate on aggregates/domain persistence.
Read-model interfaces that are purely application queries may live in Application.Contracts
or Application when that matches the existing module pattern.

### HttpApi and Web

- Controllers stay thin and delegate to Application Services.
- Razor PageModels and page scripts consume application contracts; they do not bypass the
  Application layer to access repositories.

## Public Contracts

- Preserve existing public signatures and response shapes unless all callers are traced and
  updated together.
- Existing endpoints may use `ApiResult`, `ServiceResult`, `PagedResultDto<T>`, typed DTOs,
  or the legacy `{ isSuccess, data, msg }` envelope. Match the established contract.
- A legacy public envelope belongs at the Application boundary, not in Repository methods.
- Prefer typed inputs and outputs for new internal contracts. Preserve legacy
  `Task<object>` signatures when they are part of an existing public contract.
- Never expose EF entities directly to public callers.

## Validation and Exceptions

- Validate transport/input shape in Application Services with DTO validation, guard clauses,
  or ABP validation.
- Enforce business invariants in entities/domain services.
- Validate database-specific constraints in Repository implementations where necessary.
- Use `BusinessException` or `UserFriendlyException` for expected domain/application
  failures when that matches existing project behavior.
- Let ABP's exception pipeline handle unexpected exceptions unless an established public
  response contract requires translation at the Application boundary.
- Log unexpected failures once at the boundary that has useful use-case context. Do not
  return stack traces or database details to clients.

## ABP Audit and Entity Shape

Do not require every entity table to contain the same nine audit columns. Required columns
come from the entity base class and implemented ABP interfaces.

| Entity contract | Expected persistence fields |
|---|---|
| `Entity<TKey>` | Key and declared domain fields only |
| `IHasCreationTime` / `IMayHaveCreator` | `CreationTime`, optional `CreatorId` |
| `IHasModificationTime` / `IModificationAuditedObject` | `LastModificationTime`, optional `LastModifierId` |
| `ISoftDelete` | `IsDeleted` |
| `IDeletionAuditedObject` | `DeletionTime`, optional `DeleterId` |
| `IHasExtraProperties` | `ExtraProperties` |
| `IHasConcurrencyStamp` | `ConcurrencyStamp` |
| `FullAuditedEntity<TKey>` | Creation, modification, soft-delete, and deletion audit fields |
| Audited aggregate-root variants | Matching audit fields plus aggregate-root extra properties/concurrency fields |

Before changing a migration or mapping:

1. Read the entity declaration and its inherited base classes/interfaces.
2. Compare with the EF mapping and current table.
3. Preserve every field required by that actual contract.
4. Do not add unrelated audit columns solely to satisfy a generic checklist.

## Multi-Tenancy and Soft Delete

- Prefer ABP repositories and data filters so tenant and soft-delete filters remain active.
- Do not add redundant explicit `TenantId` or `IsDeleted` predicates when verified ABP
  filters already enforce them, unless the local code intentionally uses defense in depth.
- When disabling `IDataFilter`, switching tenant context, using raw SQL, or running outside
  the normal request scope, document the reason and apply the required tenant/soft-delete
  predicates explicitly.
- Trace background jobs, integration handlers, and host-side operations because their tenant
  context may differ from normal AppService requests.

## EF Core and Repository Rules

- Use async database APIs; never use `.Result` or `.Wait()`.
- Avoid N+1 queries. Batch identifiers and query related data in one operation.
- Use projections for read models and public DTO preparation.
- Use `.AsNoTracking()` for read-only queries when tracking is not needed.
- Keep `IQueryable` inside the persistence/application query boundary; do not expose it as a
  public API contract.
- Use transactions or the ABP Unit of Work for multi-step writes that must succeed atomically.
- Avoid explicit transactions for a single `SaveChangesAsync` unless there is a demonstrated
  need.

### Raw SQL

- Parameterize every value. Never concatenate user input.
- Use existing project helpers when available:

```csharp
await WMRepositoryExtensionMethods.EnsureConnectionOpenAsync(_dbContext);
using var command = WMRepositoryExtensionMethods.CreateCommand(
    _dbContext,
    sql,
    CommandType.Text);
command.Parameters.Add(new NpgsqlParameter("projectId", projectId));
```

- Raw SQL must explicitly preserve tenant and soft-delete behavior because ABP EF filters do
  not rewrite arbitrary SQL.

## Naming and Documentation

- Methods and public types: `PascalCase`.
- Private fields: `_camelCase`.
- Locals and parameters: `camelCase`.
- Match existing sibling conventions for regions and XML documentation.
- XML documentation is valuable for public/non-obvious contracts; do not add boilerplate
  comments that merely repeat the method name.
- Preserve existing comments during edits and refactors unless the user explicitly requests
  documentation cleanup.

## Testing

Test at the layer that owns the behavior:

- Domain tests: invariants and state transitions.
- Application tests: authorization, orchestration, mapping, and response contracts.
- Repository/integration tests: EF queries, raw SQL, tenant filters, and database behavior.
- Contract tests or caller checks: public DTO/envelope compatibility.

Cover success, invalid input, not found, permission failure, tenant isolation, and expected
business failure where relevant.

## Review Checklist

- [ ] Application Service owns use-case orchestration and public mapping.
- [ ] Domain owns reusable business invariants and transitions.
- [ ] Repository contains only persistence/query behavior.
- [ ] Existing public response contract remains stable.
- [ ] Entity audit fields match its actual base class/interfaces.
- [ ] Tenant and soft-delete behavior is verified for the execution context.
- [ ] No N+1; async and projection patterns are appropriate.
- [ ] Raw SQL is parameterized and explicitly scoped.
- [ ] Tests target the layer that owns the changed behavior.

# Backend Rules (C# / ABP / EF Core)

> For refactoring, also load `REFACTOR.md`.

## Core Principle

Backend flow for repository-backed non-CRUD endpoints follows:

```text
Application Service route/authorization -> Repository processing
```

The Application Service is a thin endpoint router. The repository owns the complete use
case processing and public response construction.

## Layer Responsibilities

### Application Service

Application Services expose authorized endpoints and route calls to repositories.

- Authorize the use case with `[Authorize(...)]` or an equivalent scope check.
- Keep route attributes and public method signatures at this boundary.
- For repository-backed non-CRUD methods, directly `await` and return the matching
  repository method.
- Do not add validation, branching, mapping, `try/catch`, workflow orchestration, localization,
  or response construction to the AppService.
- Do not access `DbContext`, raw SQL, or reusable EF query construction.

Preferred shape:

```csharp
[HttpPost]
[Authorize(WorkManagementPermissions.Requirement.Update)]
public async Task<object> UpdateRequirement(InputDto input)
    => await _repository.UpdateRequirement(input);
```

### Domain

The Domain layer owns entity shape and reusable domain types.

- Entities and aggregate roots protect valid state transitions.
- Value objects model domain concepts.
- Repository interfaces describe complete use-case capabilities needed by Application
  Services.
- Domain code must not reference Application, HttpApi, Web, EF Core, or database-specific
  types.

### Repository

Repositories implement complete use cases.

- Own input/use-case validation, workflow decisions, data access, persistence, mapping,
  history hooks, notifications, external-service coordination, localization, and transaction
  coordination.
- Encapsulate EF Core, SQL, database joins, projections, batching, and persistence concerns.
- Catch expected use-case failures and construct the public object response.
- Return `Task<object>` for repository-backed non-CRUD endpoint contracts.
- Always use the exact response shape `{ isSuccess, data, msg }`; every response includes
  all three fields.
- Keep permission attributes at the AppService endpoint boundary.

Repository interfaces belong in Domain when they operate on aggregates/domain persistence.
Read-model interfaces that are purely application queries may live in Application.Contracts
or Application when that matches the existing module pattern.

### HttpApi and Web

- Controllers stay thin and delegate to Application Services.
- Razor PageModels and page scripts consume application contracts; they do not bypass the
  Application layer to access repositories.

## Public Contracts

- Repository-backed non-CRUD endpoints use `Task<object>` and return
  `{ isSuccess, data, msg }`.
- Success responses set `isSuccess = true`, return the payload in `data`, and set `msg` to
  the localized success message or an empty string.
- Failure responses set `isSuccess = false`, return an empty object/list in `data` matching
  the endpoint shape, and put the localized user-facing reason in `msg`.
- JavaScript callers read `response.isSuccess`, `response.data`, and `response.msg`.
- Do not introduce `result` or `message` aliases in new or migrated object endpoints.
- Existing `ApiResult`, `ServiceResult`, typed DTO, or `PagedResultDto<T>` contracts remain
  unchanged until the endpoint and all callers are deliberately migrated together.
- Never expose EF entities directly to public callers.

## Validation and Exceptions

- DTO attributes and ABP may reject malformed transport input before the method executes.
- Validate use-case preconditions and database-specific constraints in the repository.
- Repository methods returning object responses translate exceptions once into
  `{ isSuccess = false, data = ..., msg = localizedMessage }`.
- Do not return stack traces or database command details to clients.

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
- Application tests: authorization attributes, routes, and direct repository delegation.
- Repository/integration tests: validation, workflow, mapping, response objects, EF queries,
  raw SQL, tenant filters, and database behavior.
- Contract tests or caller checks: public DTO/envelope compatibility.

Cover success, invalid input, not found, permission failure, tenant isolation, and expected
business failure where relevant.

## Review Checklist

- [ ] Application Service contains only route/authorization and a direct awaited repository call.
- [ ] Repository owns validation, workflow, persistence, mapping, localization, and response construction.
- [ ] Repository-backed non-CRUD endpoints return all three fields `{ isSuccess, data, msg }`.
- [ ] Existing public response contract remains stable.
- [ ] Entity audit fields match its actual base class/interfaces.
- [ ] Tenant and soft-delete behavior is verified for the execution context.
- [ ] No N+1; async and projection patterns are appropriate.
- [ ] Raw SQL is parameterized and explicitly scoped.
- [ ] Tests target the layer that owns the changed behavior.

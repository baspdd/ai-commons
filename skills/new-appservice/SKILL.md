---
name: new-appservice
description: Use when adding a new ABP Application Service or feature use case that may require contracts, domain behavior, or repository access.
---

# new-appservice

Scaffolds an ABP feature service following `AI_PROMPT_BACKEND.md`. The generated flow is
Application Service orchestration -> Domain logic -> Repository data access.

## When to invoke

- User asks to "create / add / scaffold an AppService" (in any language: tạo AppService, add new service)
- User asks to add a new feature folder under `Source/src/WorkManagement.Application/<Manager>/`
- User describes a CRUD-shaped feature without specifying file layout

Do NOT invoke when the user is editing an existing AppService — use `compliance-check` instead.

## Inputs to gather (ask if missing)

1. **Entity name** (PascalCase, singular) — e.g. `Requirement`
2. **Manager folder** — e.g. `ProjectManager`, `SITManager`, `WTSManager`, `OtherManager`
3. **Business invariants/state transitions** that belong in an entity or domain manager
4. **Repository capability needed** for custom persistence/query behavior
5. **Key non-CRUD use cases**, if any (e.g. `GetByProjectId`, `CloseWithTask`)

## Procedure

1. **Load rules.** Read these once:
   - `rules/AI_PROMPT_BACKEND.md`
   - the host project's `.ai/PROJECT.md` (architecture / layer map)

2. **Verify layout** by listing/searching the target Manager folder so the new files match
   existing sibling conventions (namespace, base class, permission key shape).

3. **Define contracts and responsibilities before creating files:**

   - Application Service: authorization, request validation, orchestration, mapping, public response
   - Domain: reusable business invariants and state transitions
   - Repository: EF Core/raw SQL persistence and query behavior only

4. **Create files in dependency order:**

   a. **DTOs** under `Source/src/WorkManagement.Application.Contracts/<Manager>/<Entity>s/Dtos/`:
      - `<Entity>Dto.cs` — read shape (inherits `EntityDto<Guid>` or `FullAuditedEntityDto<Guid>`)
      - `CreateUpdate<Entity>Dto.cs` — input shape with `[Required]` / `[StringLength]` validation attributes
      - `Get<Entity>ListInput.cs` (only if list filtering needed beyond `PagedAndSortedResultRequestDto`)

   b. **Service interface** under `Source/src/WorkManagement.Application.Contracts/<Manager>/<Entity>s/`:
      - `I<Entity>AppService.cs` extending `ICrudAppService<...>`

   c. **Domain behavior** under `Source/src/WorkManagement.Domain/<Manager>/<Entity>s/`:
      - Add behavior to the entity when the invariant belongs to one aggregate
      - Add `<Entity>Manager.cs` / domain service when a rule spans entities or requires
        domain-level collaboration
      - Do not reference Application, Web, HttpApi, or EF Core types

   d. (Optional) **Repository interface** under `Source/src/WorkManagement.Domain/<Manager>/<Entity>s/`:
      - `I<Entity>Repository.cs` extending `IRepository<<Entity>, Guid>` with the custom method signatures
      - Return entities, typed query models, scalars, or collections; never an API envelope

   e. (Optional) **Repository implementation** under `Source/src/WorkManagement.EntityFrameworkCore/<Manager>/<Entity>s/`:
      - `<Entity>Repository.cs` extending `EfCoreRepository<WorkManagementDbContext, <Entity>, Guid>` and implementing `I<Entity>Repository`
      - Implement only persistence/query behavior
      - Use typed results, async EF APIs, projection/batching, and parameterized raw SQL
      - Match sibling XML-doc/region conventions; do not add boilerplate mechanically

   f. **AppService** under `Source/src/WorkManagement.Application/<Manager>/<Entity>s/`:
      - `<Entity>AppService.cs` extending `CrudAppService<...>` where CRUD is a real fit,
        otherwise inherit the local Application Service base
      - Implement `I<Entity>AppService`
      - Apply `[Authorize(WorkManagementPermissions.<Entity>.Default)]` or the matching
        use-case permission
      - Orchestrate repositories and domain behavior, then map to the established public
        response contract
      - Do not access DbContext or raw SQL directly

5. **Wire up dependencies**:
   - If creating `I<Entity>Repository`, register it in `WorkManagementEntityFrameworkCoreModule.ConfigureServices` (or confirm it's auto-registered via convention — match existing siblings).
   - If the entity is new, point out that the user still needs to add `DbSet<<Entity>>` to `WorkManagementDbContext` and create the EF Core migration / SQL migration separately.

6. **Self-review** — run the `compliance-check` skill against the files created. Fix anything before reporting done.

## Output

```
## Scaffolded <Entity> AppService

Created files:
- Source/src/WorkManagement.Application.Contracts/<Manager>/<Entity>s/Dtos/<Entity>Dto.cs
- Source/src/WorkManagement.Application.Contracts/<Manager>/<Entity>s/Dtos/CreateUpdate<Entity>Dto.cs
- Source/src/WorkManagement.Application.Contracts/<Manager>/<Entity>s/I<Entity>AppService.cs
- Source/src/WorkManagement.Application/<Manager>/<Entity>s/<Entity>AppService.cs
- (optional) Source/src/WorkManagement.Domain/<Manager>/<Entity>s/I<Entity>Repository.cs
- (optional) Source/src/WorkManagement.EntityFrameworkCore/<Manager>/<Entity>s/<Entity>Repository.cs

Next steps (not done by this skill):
- Add DbSet<<Entity>> to WorkManagementDbContext
- Create EF migration / Source/database/V*.sql migration
- Add permission key `WorkManagementPermissions.<Entity>` if missing
- Run `dotnet build` and address any errors
```

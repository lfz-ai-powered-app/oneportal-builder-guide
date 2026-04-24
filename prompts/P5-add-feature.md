# P5 — Add a Feature End-to-End

**When to use:** every time you add a new feature. Re-use this prompt.

**What it produces:** domain entity, EF config, migration, CQRS handler, validator, API controller, Blazor page, one unit test.

---

## Paste this into Claude Code

```
Read CLAUDE.md at the repo root. Follow every rule.

PortalName: <PortalName>

FEATURE DESCRIPTION (plain English):
"""
<describe your feature here — what entity, what operations (list/create/update/delete), what permission it requires>

Example:
"A list of warehouse locations. Each location has code, name, address, and active flag. Users can list, create, edit, and deactivate locations. Requires the 'manage-locations' permission."
"""

Produce all of the following, each in its correct layer, and explain the dependency direction as you go:

## Domain layer — <PortalName>.Domain

- Create the entity class in `<PortalName>.Domain/Entities/`. Only business invariants and domain methods. No EF attributes, no System.ComponentModel.DataAnnotations.
- If the feature needs a domain service or value object, add it in `<PortalName>.Domain/`.
- NO references to EF Core, MediatR, ASP.NET, or AutoMapper.

## Application layer — <PortalName>.Application

- Create DTOs in `<PortalName>.Application/Features/<Feature>/Dtos/`.
- Create MediatR requests in `<PortalName>.Application/Features/<Feature>/Commands/` and `Queries/`:
  - One `ListXxxQuery` returning a paginated list DTO.
  - One `GetXxxByIdQuery` returning a single DTO.
  - `CreateXxxCommand`, `UpdateXxxCommand`, `DeleteXxxCommand` as the feature needs.
- Create handlers for each in the same folder.
- Create FluentValidation validators in `<PortalName>.Application/Features/<Feature>/Validators/`. Every command MUST have a validator.
- If the Application needs a repository, define the interface here (e.g. `IXxxRepository`) — Infrastructure will implement it.

## Infrastructure layer — <PortalName>.Infrastructure

- Add a `DbSet<Entity>` to the DbContext.
- Add an IEntityTypeConfiguration class under `<PortalName>.Infrastructure/Persistence/Configurations/`. Define keys, indexes, field lengths, relationships.
- Generate an EF migration:
  `dotnet ef migrations add Add<Feature> --project <PortalName>.Infrastructure --startup-project <PortalName>.API`
- If you defined a repository interface in Application, implement it here.

## API layer — <PortalName>.API

- Create `<PortalName>.API/Controllers/<Feature>Controller.cs`:
  - `[ApiController]`, `[Route("api/<feature>")]`.
  - Every action must be `[Authorize(Policy = "<specific-policy>")]` or explicitly `[AllowAnonymous]`.
  - Actions delegate to MediatR via the `ApiControllerBase` pattern.
  - Return `ApiResponse<T>` envelope (match StoreTrack's pattern if `ApiResponse` exists in Application).
- If the feature needs a new authorization policy, add it in `<PortalName>.Infrastructure/DependencyInjection.cs` under the policy registrations.

## Web layer — <PortalName>.Web

- Create `<PortalName>.Web/Pages/<Feature>Page.razor`:
  - `@page "/<feature>"`.
  - `@attribute [Authorize(Policy = "<same-policy>")]`.
  - Inject `IHttpClientFactory` and use the `"<PortalName>Api"` named client.
  - Show a loading state, an error state, and an empty state.
  - Use Bootstrap Icons for action icons; style with var(--...) tokens only.
  - Use the shared `<LfzSpinner />` for async loads.
- Add the page to `NavMenu.razor` inside an AuthorizeView that checks the same policy.

## Test layer — <PortalName>.Tests.Unit (create if missing)

- Create an xUnit project `<PortalName>.Tests.Unit.csproj` if it doesn't exist. Reference `<PortalName>.Application` and `<PortalName>.Domain`.
- Write ONE happy-path test per command/query handler. Use an in-memory repository stub or a fake `IApplicationDbContext`.
- Write ONE validator test covering the "invalid input" branch for each validator.

## After generation

1. Run `dotnet build` and fix warnings.
2. Run `dotnet test`. Every test green.
3. Run `dotnet ef database update` to apply the new migration.
4. Start the API and Web, navigate to the new page, and do a manual smoke test.
5. Print a summary listing every file created/modified with a one-sentence purpose each.

Do NOT:
- Put EF annotations on Domain entities.
- Put business logic in the API controller.
- Skip the validator.
- Leave the `[Authorize]` attribute off.
- Use `DateTime.Now` — always UTC.
- Catch Exception without logging and rethrowing.
- Introduce new NuGet packages without explaining why in the commit message.
```

---

## Post-generation review

Go through this list on the diff before you commit:

- [ ] Dependencies point inward. No `using Microsoft.EntityFrameworkCore` in Domain. No `using Microsoft.AspNetCore` in Application.
- [ ] Every endpoint has `[Authorize(Policy = "...")]` or explicit `[AllowAnonymous]`.
- [ ] Every command/query has a validator and at least one test.
- [ ] Every user-visible string is in the Razor markup, not hardcoded into handlers.
- [ ] No hex colours in markup.
- [ ] New migration has a descriptive name (`Add<Feature>`).
- [ ] No secrets or connection strings committed.

## Next

Keep running P5 for each new feature. When done for the day, run **P6 — Commit & Open PR**.

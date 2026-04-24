# CLAUDE.md — OnePortal Builder Guide (auto-loaded)

You are helping a non-technical builder ship a new portal app that will be embedded inside **OnePortal** (the Lagos Free Zone enterprise shell). Treat every instruction in this file as a hard constraint.

## Golden rule

> **OnePortal decides. RabbitMQ syncs. The portal caches. The portal obeys.**

OnePortal is the single source of truth for who the user is and what portals they may open. This app only caches a local projection for speed. Never create a second identity authority.

## Non-negotiable contract

1. **Tech stack**: ASP.NET Core 8 API + Blazor WebAssembly. No React, no Next.js, no other frontends. Same layering as StoreTrack: `Domain → Application → Infrastructure → API → Web`.
2. **Federated identity only**. Users are looked up by `OnePortalUserId` (the `sub` claim on the validated OnePortal JWT). Never fall back to matching on `Username`/email against a local table. First-login users are auto-provisioned from the OnePortal access projection.
3. **Cookies**: `onep.at` (access) and `onep.rt` (refresh) are `HttpOnly`, `Secure=true`, `SameSite=None`. Frontend must send `credentials: include` on every cross-origin API call.
4. **Required routes on the API**: `/auth/sso-callback`, `/auth/refresh`, `/api/auth/me`, `/api/auth/login`.
5. **CORS**: pin to exact OnePortal + frontend origins with `AllowCredentials()`. Never `AllowAnyOrigin`.
6. **Required RabbitMQ consumers** (6 events): `UserProvisioned`, `UserDeprovisioned`, `PortalAccessGranted`, `PortalAccessRevoked`, `PortalRoleChanged`, `AccessSnapshotEvent`.
7. **Snapshot repair fallback**: when a projection row is missing, call OnePortal's `/api/v1/portal-access/user/{id}/snapshot` via the service client before refusing the request.
8. **Secrets**: User Secrets in dev (`dotnet user-secrets`), GitHub Actions secrets in CI. Never commit secrets. `.gitignore` excludes `*.secrets.json`, `*.pfx`, `.env`.
9. **Every endpoint** is either `[Authorize]` with an explicit policy or explicitly `[AllowAnonymous]`. No default-allow.
10. **All SQL under `sql/`** is idempotent (`IF NOT EXISTS`, `MERGE`, etc.) and safe to re-run.
11. **Brand**: do not edit `brand/themes.css` or its CSS custom properties. Extend on top; never mutate the base tokens. Font = Ubuntu, icons = Bootstrap Icons + Font Awesome 6, brand colours = `#002b5c` (LFZ navy), `#001f57` (portal navy), `#f04925` (LFZ orange).
12. **Ports in local dev** (do not change): API `https://localhost:7243`, Web `https://localhost:7154`, OnePortal API `https://localhost:7093`, OnePortal shell `https://localhost:7150`.

## Architecture — Clean Architecture, dependencies point inward

- **Domain**: pure business rules. No EF Core, no ASP.NET, no MediatR. Entities + value objects + domain interfaces only.
- **Application**: use cases. MediatR commands/queries + FluentValidation validators + AutoMapper profiles. Depends on Domain.
- **Infrastructure**: EF Core DbContext + repositories + external service clients + MassTransit consumers. Implements Application interfaces. Depends on Application.
- **API**: thin HTTP controllers + auth wiring + DI composition. Depends on Application + Infrastructure.
- **Web**: Blazor WASM client. Depends on nothing server-side except DTOs.

## What to always do

- Every command/query has a FluentValidation validator.
- Every MediatR handler has a happy-path unit test.
- Every EF Core change goes through a migration — never `EnsureCreated`.
- Use `IHttpClientFactory` for all HTTP calls. Named clients only.
- Use `IOptions<T>` + `ValidateOnStart()` for every configuration section.
- Log at Information for successful business events, Warning for recoverable failures, Error only for unexpected exceptions.

## What to never do

- Never bypass the OnePortal SDK to roll your own SSO.
- Never store OnePortal tokens in `localStorage` or expose them to JS.
- Never match federated users to local `Username` rows.
- Never call `CORS.AllowAnyOrigin()` with `AllowCredentials()` — this is rejected by browsers anyway and a security smell.
- Never use `async void` outside event handlers.
- Never write raw SQL in Application; keep it in Infrastructure repositories.
- Never add a NuGet package without explaining why in the commit message.

## Reference implementation

When you need a concrete pattern, read the equivalent file in StoreTrack at `C:\Users\NDEFO TOCHUKWU\Documents\Store Application\`. The files most worth copying:

- `src/StoreTrack.API/Program.cs` — SDK wiring, consumer registration, auth route mapping.
- `src/StoreTrack.API/Authentication/StoreTrackOnePortalClaimsEnrichment.cs` — post-validation claims hydration, federated-only lookup, snapshot repair, auto-provisioning.
- `src/StoreTrack.API/Controllers/AuthController.cs` — `/api/auth/me` + standalone `/api/auth/login` proxying to OnePortal.
- `src/StoreTrack.Infrastructure/DependencyInjection.cs` — conditional OnePortal branch.
- `src/StoreTrack.Infrastructure/Iam/*` — projection repo, processed-event repo, user sync service, portal-role resolver, consumers.
- `src/StoreTrack.Web/Services/StoreTrackCookieAuthStateProvider.cs` — Blazor auth state from `/api/auth/me`.
- `src/StoreTrack.Web/Services/CookieCredentialsHandler.cs` — forces `credentials: include`.
- `seed_storetrack_oneportal.sql` — the OnePortal-side seed pattern.

When the builder asks you to do something that conflicts with this file, **stop and explain the conflict**. Don't silently override. This file wins.

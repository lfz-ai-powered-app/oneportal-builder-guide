# P3 — Wire OnePortal Authentication

**When to use:** after P2. This is the biggest prompt in the guide. Review the diff carefully before committing.

**What it produces:** a fully wired federation flow (SSO iframe + standalone login), projection repositories, 6 IAM event consumers, snapshot-repair fallback, CORS, cookie handling on both API and Web.

---

## Paste this into Claude Code

```
Read CLAUDE.md at the repo root. Every rule in it is a hard constraint. Then read
C:\Users\NDEFO TOCHUKWU\Documents\Store Application\src\StoreTrack.API\Authentication\StoreTrackOnePortalClaimsEnrichment.cs
as your reference for the claim-enrichment hook. Copy the PATTERN, not the code — adapt names to this project.

Portal code: <PORTAL_CODE>
PortalName: <PortalName>

Do the following in order.

## 1. Install OnePortal SDK packages

Add these NuGet references to <PortalName>.API.csproj and <PortalName>.Infrastructure.csproj (whichever layer actually uses each):

- OnePortal.Integration
- OnePortal.Integration.Auth
- OnePortal.Integration.Authorization
- OnePortal.Integration.Contracts

Version is controlled by $(OnePortalSdkVersion) in Directory.Build.props.
Also add MassTransit and MassTransit.RabbitMQ to <PortalName>.Infrastructure.csproj.

## 2. Wire OnePortal in <PortalName>.API/Program.cs

Match the StoreTrack pattern:

- Read `Authentication:Mode` from config; default "Local".
- If OnePortal mode, require OnePortal:Messaging:ConnectionString unless RequireBroker=false.
- Call `builder.Services.AddOnePortalIntegration(builder.Configuration).WithServiceClient()`.
- Call `.WithEventConsumers([typeof(UserProvisionedConsumer).Assembly], bus => { ... })`:
  - If connection string is empty → `bus.UsingInMemory(...)` (dev fallback).
  - Otherwise → `bus.UsingRabbitMq(...)` with the connection string.
- Register `<PortalName>OnePortalClaimsEnrichment` as `IPostConfigureOptions<JwtBearerOptions>`.
- Call `app.UseOnePortalIntegration()` in the pipeline when OnePortal mode is on.
- Map the four required routes under an `AllowAnonymous` group named `/auth`:
  - GET /auth/login          → SsoLoginService.HandleAsync
  - GET /auth/sso-callback   → SsoAssertionHandler.HandleAsync
  - GET /auth/refresh        → SilentRefreshHandler.HandleAsync
- Map a GET `/` handler that redirects to `StoreTrack:FrontendUrl` (use the config key `<PortalName>:FrontendUrl`).

## 3. CORS

Register `"AllowBlazor"` CORS policy:
- Read `Cors:Origins` from config.
- `.WithOrigins(origins).AllowAnyHeader().AllowAnyMethod().AllowCredentials()`.
- Default origins for Development: https://localhost:7150, https://localhost:7154, http://localhost:5021, https://localhost:7093.

## 4. Projection repositories in <PortalName>.Infrastructure

Under `<PortalName>.Infrastructure/Iam/`:

- `AccessProjectionRepository` implementing `IAccessProjectionRepository`. Upsert uses (UserId, PortalCode) as the unique index. Do NOT write to rows whose PortalCode isn't our configured one.
- `ProcessedEventRepository` implementing `IProcessedEventRepository`. Primary key is EventId.
- `IamUserSyncService` that takes an `AccessProjection` and upserts a local `User` by `OnePortalUserId`. Never matches by Username.
- `PortalRoleResolutionService` that joins `PortalRoleDefinitions` + `PortalRolePermissions` for a given portal role code, returning `(appRoleName, IReadOnlyList<string> permissions)`.
- `OnePortalUserProfileFetcher` that calls OnePortal's `/api/v1/users/{id}` via the service client and syncs optional profile fields on the local User.

Register all of these in `<PortalName>.Infrastructure/DependencyInjection.cs` inside the `if (authMode == "OnePortal")` branch.

## 5. EF DbContext changes

Ensure the DbContext has DbSets for:
- IamAccessProjections
- ProcessedIamEvents
- PortalRoleDefinitions
- PortalRolePermissions

Add a migration `AddOnePortalIamSsot` if the tables aren't already there.
Unique index: IamAccessProjections(UserId, PortalCode).
Unique index: PortalRolePermissions(PortalRoleCode, Permission).
PK on ProcessedIamEvents.EventId.

Also add these columns to the local `Users` table (adapt if you use a different aggregate for users):
- OnePortalUserId (int, nullable, unique-filtered index)
- OnePortalPortalRoleCode (nvarchar(120), nullable)
- OnePortalProfileSyncedUtc (datetimeoffset, nullable)

## 6. Claim enrichment

Create `<PortalName>.API/Authentication/<PortalName>OnePortalClaimsEnrichment.cs`. Behaviour (copy from StoreTrack's reference file):

- On TokenValidated, resolve the `sub` claim and look up the local User by OnePortalUserId. Federated-only — NEVER fall back to Username/email matching.
- Fetch `AccessProjection` from the local repo. If missing and `OnePortal:Projection:EnableBootstrapFallback` is true, call `OnePortalApiClient.GetUserAccessAsync(sub, ct)` and upsert.
- If still missing and token claims contain `portal_role` + `email`, synthesize a projection from the token and upsert (dev fallback only).
- If the projection is inactive but we can reactivate from token claims in bootstrap mode, do so and log.
- If the local User doesn't exist and we have an active projection, auto-provision via IamUserSyncService.ApplyProjectionToUserAsync.
- Resolve the portal role → (appRole, permissions) via PortalRoleResolutionService.
- Build a fresh ClaimsIdentity using the JwtBearer auth type; replace context.Principal.
- In the catch block, treat OperationCanceledException (when ctx.HttpContext.RequestAborted is cancelled) as a Debug log and DO NOT call context.Fail. Any other exception → Error log + context.Fail("...").

## 7. AuthController

Create `<PortalName>.API/Controllers/AuthController.cs` with:

- `[HttpGet("me")]` `[Authorize]` returning the authenticated principal's key claims as a DTO (userId, name, role, permissions[]). Return NotFound if not in OnePortal mode.
- `[HttpPost("login")]` `[AllowAnonymous]` that proxies email+password to OnePortal's `/api/auth/login`, handles MFA/password-change/error cases, and sets the `onep.at` / `onep.rt` cookies on success with:
    HttpOnly = true
    Secure = true
    SameSite = SameSiteMode.None
    Path = "/"
- `[HttpGet("logout")]` `[AllowAnonymous]` that deletes both cookies and redirects to a safe return URL (must match FrontendUrl scheme/host/port).

## 8. Blazor Web wiring

In `<PortalName>.Web/Program.cs`:

- Read `Authentication:Mode` and `ApiBaseUrl`.
- If OnePortal mode:
  - Register `CookieCredentialsHandler` (forces credentials:include on every request).
  - Add a named HttpClient `"<PortalName>Api"` with BaseAddress = ApiBaseUrl and the CookieCredentialsHandler attached.
  - Register `<PortalName>CookieAuthStateProvider` and bind it as the AuthenticationStateProvider.
- Call AddAuthorizationCore.

`<PortalName>.Web/Services/CookieCredentialsHandler.cs`:
Single DelegatingHandler override that calls `request.SetBrowserRequestCredentials(BrowserRequestCredentials.Include)` and delegates.

`<PortalName>.Web/Services/<PortalName>CookieAuthStateProvider.cs`:
AuthenticationStateProvider override that GETs `/api/auth/me` via the named HttpClient, maps claims, returns Anonymous on failure. Expose `NotifyAuthenticationStateChanged()`.

## 9. Blazor login page

`<PortalName>.Web/Pages/Login.razor`:
- If OnePortal mode: two buttons — "Sign in with OnePortal" (posts to /api/auth/login) and "Open from OnePortal Shell" (navigates to {ApiBaseUrl}/auth/login?returnUrl={frontendUrl}).
- Refresh the auth state + navigate to "/" on success.

## 10. Smoke test checklist

Print at the end:

- [ ] `dotnet build` clean
- [ ] 6 MassTransit consumers registered (dump them via reflection at startup)
- [ ] `GET /api/auth/me` returns 401 when no cookie, 200 when signed in
- [ ] Cookies `onep.at` + `onep.rt` present after SSO exchange, both HttpOnly + Secure + SameSite=None
- [ ] CORS origins pinned (no AllowAnyOrigin)
- [ ] Claim enrichment cancellation path returns no context.Fail

Do NOT:
- Expose any token to JavaScript.
- Store tokens in localStorage or sessionStorage.
- Add legacy Username fallback.
- Add an "allow any origin" CORS policy.
- Skip ValidateOnStart on OnePortalJwtOptions.
```

---

## Troubleshooting

**`JWT challenge triggered for ... Error: null`**
Your Authorization header is empty. Two causes seen before:
1. `ServiceTokenProvider.TokenResponse` record missing `[JsonPropertyName("access_token")]` + `[JsonPropertyName("expires_in")]` — deserialization silently produces null.
2. `OnePortal.Application.OAuth.Dtos.ClientCredentialsTokenRequest` PascalCase properties not matching snake_case form fields — controller parameters should bind `[FromForm(Name="grant_type")]` etc.

Both fixes live in OnePortal's repo, not yours — escalate to the integrator.

**`HTTP POST /api/v1/sso/exchange responded 200 in 17000 ms`**
OnePortal cold-start. Caused by running under the debugger (F5). Use **Ctrl+F5** (Start Without Debugging). If still slow: `ALTER DATABASE [<OnePortalDb>] SET AUTO_CLOSE OFF` — ask the integrator.

**`TaskCanceledException at ClaimsEnrichment line XX`**
The client aborted the request (Blazor re-auth churn or `NavigateTo(forceLoad:true)` race). Your catch block should treat this as Debug + skip `context.Fail`. If it doesn't, your code drifted from the pattern — re-read step 6 above.

**Cookies not sent on `/api/auth/me`**
Check the three-origins contract: API 7243, Web 7154, OnePortal shell 7150. Any scheme/host/port drift breaks cookies. Also enable third-party cookies in Chrome for localhost during dev.

## Verification

```bash
dotnet build
dotnet run --project <PortalName>.API
# separate terminal
dotnet run --project <PortalName>.Web
```

Open OnePortal shell at `https://localhost:7150`, sign in, click your portal tile, and walk through the DevTools Network checklist in README §9.

## Next

Run **P4 — Apply LFZ Brand**.

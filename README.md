# OnePortal Builder Guide

> The organizational standard for building apps that embed inside OnePortal.
> Written for non-technical staff using Claude Code.

If you follow this guide top to bottom, you will end with an app that runs locally, passes the integrator's review on the first try, and looks native inside OnePortal.

---

## 1. What You're Building & Who This Is For

You are building a **mini-app** (a "portal") that will appear as a tile inside **OnePortal** — the Lagos Free Zone enterprise shell. When a user clicks your tile, your app loads inside an iframe. To the user it looks like part of OnePortal. Behind the scenes:

- **OnePortal** decides *who* the user is and *whether they may open your portal*.
- **Your app** decides *what they can do once they're inside*.

You do not need to write authentication from scratch. OnePortal does it for you. You just call a few standard endpoints, trust the cookies OnePortal sets, and focus on your business feature.

**You are a good fit to build this if you can:** install Visual Studio, follow numbered steps, paste prompts into Claude Code, and read a checklist. You do not need to be a senior developer.

## 2. Prerequisites — Install Once

Install these before you start. Each is a standard installer.

| Tool | Why | Where |
|---|---|---|
| Visual Studio 2022 (Community edition is free) | IDE to run & debug your app | https://visualstudio.microsoft.com/downloads/ |
| .NET 8 SDK + "ASP.NET and web development" workload | Compiles the app | Via the VS installer |
| SQL Server Express or LocalDB | Your app's database | Via the VS installer |
| SQL Server Management Studio (SSMS) | Runs SQL scripts | https://aka.ms/ssms |
| Git for Windows | Source control | https://git-scm.com/download/win |
| GitHub CLI (`gh`) | Push code & open PRs | https://cli.github.com/ |
| Claude Code | AI pair programmer | https://claude.com/claude-code |
| Node.js LTS | Only needed for `degit` to clone the template | https://nodejs.org/ |

**Accounts you'll need from the integrator:**

1. Membership in the `LFZ-OnePortal` GitHub organization.
2. A dev **OnePortal service client ID + secret** for your portal (e.g. `storetrack_portal_client_...`).
3. Access to the OnePortal dev database (only the integrator writes to it; you read your own seed output to confirm).
4. Read access to the private NuGet feed hosting the `OnePortal.Integration.*` packages.

Run this once after installing Git + GitHub CLI:

```bash
gh auth login
dotnet dev-certs https --trust
```

## 3. Clone the Starter Template

Pick a portal code in CAPS, no spaces — e.g. `ASSETTRACK`, `FLEETOPS`. This is your **PortalCode**. Use it everywhere.

```bash
npx degit LFZ-OnePortal/oneportal-builder-guide/template my-portal-app
cd my-portal-app
```

Open `prompts/P1-scaffold.md`, copy the prompt into Claude Code, replace `<Portal>` with your chosen name, and paste. Claude Code will:

- Rename all `<Portal>` placeholders to your name.
- Set up the solution file and project references.
- Initialize a Git repository.
- Create a blank EF migration.
- Print the port numbers it picked (keep the defaults unless you have a reason).

Open the resulting `.sln` in Visual Studio. You should see **five projects**:

| Project | What it is |
|---|---|
| `MyPortal.Domain` | Your business rules — pure C# classes, no frameworks. |
| `MyPortal.Application` | Your use cases — MediatR commands/queries + validators. |
| `MyPortal.Infrastructure` | Database, external services, message consumers. |
| `MyPortal.API` | HTTP endpoints — talks to the outside world. |
| `MyPortal.Web` | What the user sees — Blazor WebAssembly. |

## 4. Architectural Style — Clean Architecture in Plain English

Clean Architecture means **layers that depend only on the layers beneath them**.

```
Web ──► API ──► Application ──► Domain
          │           ▲
          └──► Infrastructure
                      │
                      └───────────┘  (implements Application interfaces)
```

**Rules:**

- Domain has no references to anything. It's your business rules.
- Application references Domain. It defines *what* the app does (use cases) and *what interfaces* it needs.
- Infrastructure references Application. It *implements* those interfaces — EF Core, HTTP clients, message consumers.
- API references Application + Infrastructure. It's the HTTP layer that wires everything together.
- Web references nothing server-side except DTOs. It's a separate browser app.

**Why this matters for you:** if a teammate or an auditor reviews your code, they will trace dependencies. Anything pointing the wrong way is a red flag. Stick to the layering and the review goes smoothly.

## 5. Set Up Your Local Database & Run Scripts

Two databases are involved:

- **Your local DB** (lives on your machine) — holds your app's own data.
- **OnePortal dev DB** (lives on the integrator's server) — you seed a row here so OnePortal recognizes your portal.

**Step 5.1 — Your local DB**

1. Open SSMS, connect to `(localdb)\MSSQLLocalDB` or `.\SQLEXPRESS`.
2. Create a database named `<YourPortal>Db` (right-click Databases → New Database).
3. Open `sql/01-local-db.sql` and run it against the DB you just created. This adds the OnePortal projection tables (`IamAccessProjections`, `ProcessedIamEvents`, `PortalRoleDefinitions`, `PortalRolePermissions`).
4. In `MyPortal.API/appsettings.Development.json`, set `ConnectionStrings:DefaultConnection` to point at that DB.
5. In a terminal: `dotnet ef database update --project MyPortal.Infrastructure --startup-project MyPortal.API`. This applies your EF migrations on top of the OnePortal tables.

Open `prompts/P2-db-seed.md` in Claude Code if any of the above needs automation — it will fill in the connection strings and generate your per-portal OnePortal seed.

**Step 5.2 — Register your portal in OnePortal**

Open the generated `sql/02-seed-oneportal.sql`. Check the values (portal code, roles, pilot email). Send this script to the integrator — only they run it against the OnePortal dev DB. Do **not** run this yourself unless the integrator tells you to.

Why separate: OnePortal is shared infrastructure. Only one person writes to it.

## 6. Wire OnePortal Authentication

Open `prompts/P3-wire-auth.md` in Claude Code. It will:

- Install `OnePortal.Integration`, `OnePortal.Integration.Auth`, `OnePortal.Integration.Authorization`, `OnePortal.Integration.Contracts` from the private NuGet feed.
- Wire `AddOnePortalIntegration().WithServiceClient().WithEventConsumers(...)` in `MyPortal.API/Program.cs`.
- Map the four required routes: `/auth/sso-callback`, `/auth/refresh`, `/api/auth/me`, `/api/auth/login`.
- Implement `IAccessProjectionRepository` and `IProcessedEventRepository` against your local DB.
- Write `MyPortalOnePortalClaimsEnrichment.cs` — the hook that runs after each JWT is validated to turn OnePortal claims into your app's local claims. Federated-only: the user is always looked up by `OnePortalUserId`.
- Add MassTransit consumers for the six IAM events.
- Configure CORS pinned to `https://localhost:7150` (OnePortal shell) and your frontend origin.
- Wire `CookieCredentialsHandler` in `MyPortal.Web/Program.cs` so all fetches carry cookies.

**What's happening under the hood:** When a user clicks your tile in OnePortal, OnePortal mints a one-time assertion JWT, puts it in the iframe URL, and your API's `/auth/sso-callback` exchanges it for session cookies. All subsequent requests flow via those cookies. You never store tokens in JavaScript.

**Three-origins rule** (memorize this):

| Origin | Port (dev) | Role |
|---|---|---|
| OnePortal shell | `7150` | Where the user lives |
| Your API | `7243` | Sets `onep.at` / `onep.rt` cookies |
| Your Web | `7154` | Calls `/api/auth/me` with cookies |

If any port/origin drifts, auth breaks silently. Keep these defaults.

## 7. Apply the OnePortal (LFZ) Brand

Open `prompts/P4-apply-brand.md`. It will:

- Copy `brand/themes.css` and `brand/app.css` verbatim into `MyPortal.Web/wwwroot/css/`.
- Add the Ubuntu font, Bootstrap Icons, and Font Awesome 6 CDN links to `index.html`.
- Wire the `.dark-theme` class toggle on `<html>` (JS helper).
- Add a `MainLayout.razor` with the LFZ navy header + sidebar pattern.

**Never edit the base CSS variables.** Extend on top. When LFZ refreshes the brand, the integrator will publish a new `themes.css` and every portal picks it up without touching component code.

**Brand cheat-sheet:**

| Token | Value |
|---|---|
| LFZ navy | `#002b5c` |
| Portal navy | `#001f57` |
| LFZ accent orange | `#f04925` |
| Font | Ubuntu 400/500/700 |
| Icons | Bootstrap Icons 1.11.3, Font Awesome 6.5.0 |
| Theme switch | add `.dark-theme` to `<html>` |

## 8. Build Your First Feature End-to-End

Open `prompts/P5-add-feature.md`. Describe one feature in plain language (e.g. "a list of warehouse locations with create, edit, delete"). Claude Code will produce:

- A Domain entity + EF configuration.
- An EF migration.
- MediatR commands/queries + handlers.
- FluentValidation validators.
- An `[Authorize]`-gated API controller.
- A Blazor page that consumes the API via the named HttpClient.
- A happy-path unit test.

**What you should review before running it:**

- [ ] Does the Domain entity have only business rules (no EF attributes, no DTOs)?
- [ ] Is every input validated?
- [ ] Is every endpoint `[Authorize(Policy = "...")]` or explicitly `[AllowAnonymous]`?
- [ ] Does the Blazor page show a sensible loading & error state?

## 9. Run & Test Locally in Visual Studio

1. In Solution Explorer, right-click the solution → **Configure Startup Projects** → **Multiple startup projects** → set `MyPortal.API` and `MyPortal.Web` both to **Start**.
2. Press **F5**. Two browser tabs should open on `https://localhost:7243` (API Swagger) and `https://localhost:7154` (your Blazor app standalone).
3. Close them. The real test is via OnePortal.
4. Make sure the OnePortal API (`https://localhost:7093`) and OnePortal shell (`https://localhost:7150`) are running — ask the integrator if you don't have them locally.
5. Open OnePortal shell, log in with your pilot account, and click your portal tile.
6. Open DevTools → Network tab. Walk through this sequence:

| Step | Expected |
|---|---|
| `/auth/sso-callback?assertion=…` | **302** redirect |
| `/` (your API root) | **302** to your frontend |
| `/api/auth/me` | **200** with your user info |
| Cookies in DevTools → Application | `onep.at` + `onep.rt` present, `HttpOnly` ✓, `Secure` ✓, `SameSite=None` |

If anything fails, check `prompts/P3-wire-auth.md` troubleshooting section and the StoreTrack reference runbook.

**Run the tests:**

```bash
dotnet test
```

All green before you move on.

## 10. Security Checklist Before You Commit

Tick every box. This is the reviewer's first check.

- [ ] Cookies are `HttpOnly`, `Secure=true`, `SameSite=None`.
- [ ] No secrets committed (`git log -p | grep -iE "password|secret|key"` returns nothing).
- [ ] `appsettings.Development.json` uses LocalDB or integrated auth only — no real credentials.
- [ ] Dev secrets live in User Secrets (`dotnet user-secrets list` shows them).
- [ ] CORS is pinned to exact origins with `AllowCredentials()`. No `AllowAnyOrigin`.
- [ ] Every endpoint has `[Authorize]` or explicit `[AllowAnonymous]`.
- [ ] Every command/query has a FluentValidation validator.
- [ ] No legacy-username lookup — identity is always `OnePortalUserId`.
- [ ] `OnePortal:Projection:EnableBootstrapFallback` is `true` in Development and `false` in base `appsettings.json`.
- [ ] All SQL scripts under `sql/` are idempotent and re-runnable.
- [ ] `.gitignore` excludes `*.secrets.json`, `*.pfx`, `.env`, `bin/`, `obj/`.

## 11. Commit, Push & Open a PR

Open `prompts/P6-commit-and-pr.md` in Claude Code. It will:

- Create a feature branch named `feat/<short-name>`.
- Stage your changes.
- Write a conventional commit message.
- Push via `gh`.
- Open a PR against `LFZ-OnePortal/<your-repo>` using the PR template.

**Branch naming:**

| Prefix | When |
|---|---|
| `feat/` | New feature |
| `fix/` | Bug fix |
| `chore/` | Housekeeping (deps, configs) |
| `docs/` | Documentation only |

**Conventional commit examples:**

```
feat(locations): add create/edit/delete for warehouse locations
fix(auth): remove legacy username fallback; lookup by OnePortalUserId only
chore(ci): add secrets scan to PR workflow
```

**Never** force-push to `main`. **Never** commit to `main` directly.

## 12. Hand Off to the Integrator

When the PR is open and CI is green, message the integrator with:

1. **PR link**.
2. The path to your `sql/02-seed-oneportal.sql` (the integrator runs this against the OnePortal dev DB).
3. **Screenshots** — one of your app running standalone, one of it running inside the OnePortal iframe.
4. Confirmation that **every checkbox in the PR template is ticked**.

The integrator will:

- Review the diff.
- Run your seed against OnePortal.
- Register your `PortalClient` row.
- Merge the PR and deploy to staging.

If anything fails review, the feedback goes back to you as a PR comment. Fix, push, reviewer re-checks.

---

## Quick reference

| You need to… | Go to |
|---|---|
| Start a new portal app | [prompts/P1-scaffold.md](prompts/P1-scaffold.md) |
| Set up local DB | [prompts/P2-db-seed.md](prompts/P2-db-seed.md) |
| Wire OnePortal auth | [prompts/P3-wire-auth.md](prompts/P3-wire-auth.md) |
| Apply the LFZ brand | [prompts/P4-apply-brand.md](prompts/P4-apply-brand.md) |
| Add a feature | [prompts/P5-add-feature.md](prompts/P5-add-feature.md) |
| Commit & open PR | [prompts/P6-commit-and-pr.md](prompts/P6-commit-and-pr.md) |
| See a working reference | StoreTrack — the first integrated portal |
| Read the authoritative spec | `ONEPORTAL_INTEGRATION_STANDARD.html` in the OnePortal repo |

## Trouble?

First stop: the "Troubleshooting" appendix in `prompts/P3-wire-auth.md`. Most problems are one of:

1. Ports don't match (keep 7093/7150/7243/7154).
2. Browser blocks third-party cookies — enable them for localhost while developing.
3. OnePortal seed not run — ask the integrator.
4. Running under the debugger makes first request take 15+ s — use **Ctrl+F5** (Start Without Debugging) for testing.

Ask the integrator if you're stuck for more than 30 minutes. This guide is a living document — your confusion is a bug we want to fix.
#   o n e p o r t a l - b u i l d e r - g u i d e  
 
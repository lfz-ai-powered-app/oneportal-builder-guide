# Template Setup — For the Integrator (One-Time)

You're reading this because you want to stop hand-scaffolding a fresh project every time someone new wants to build an app. This guide turns the `template/` folder in this repo into a **GitHub Template Repository**, so from then on every builder can create their own full scaffold with one click — no terminal, no `degit`, no git surgery.

You do this **once**. Budget 15 minutes.

---

## What you're creating

Two GitHub repos, one role each:

| Repo | Who uses it | What it is |
|---|---|---|
| `lfz-ai-powered-app/oneportal-builder-guide` (this repo) | Everyone | The guide (`index.html`, `prompts/`, `brand/`, this doc) |
| `lfz-ai-powered-app/portal-template` (you create this) | Builders, once per new app | The Clean Architecture scaffold marked as a GitHub Template Repository |

After setup, a builder's flow is:

1. Open `https://github.com/lfz-ai-powered-app/portal-template` in a browser.
2. Click **Use this template** → **Create a new repository**.
3. Pick a name (`my-portal-app`).
4. GitHub creates `https://github.com/lfz-ai-powered-app/my-portal-app` with the full scaffold and a clean commit history.
5. They clone *their own* repo in Visual Studio (Step 2 of the builder guide) and start.

No one other than you ever creates scaffolds by hand.

---

## One-time setup — do this now

### 1. Create the empty GitHub repo

Any of these work; pick one:

**With the GitHub CLI** (fastest):

```bash
gh repo create lfz-ai-powered-app/portal-template \
  --private \
  --description "Clean Architecture starter for OnePortal apps. Click 'Use this template'." \
  --confirm
```

**Via the browser**: go to <https://github.com/organizations/lfz-ai-powered-app/repositories/new>, name it `portal-template`, set visibility to **Private** (or Internal if your plan supports it), and don't add a README / .gitignore / license — we'll push them ourselves.

### 2. Push the template contents

From this repo's root, push the `template/` folder as a standalone repo:

```bash
# From .../oneportal-builder-guide
cd template

git init
git branch -M main
git add .
git commit -m "chore: initial scaffold"
git remote add origin https://github.com/lfz-ai-powered-app/portal-template.git
git push -u origin main
```

Check the repo on GitHub. You should see `Portal.Domain/`, `Portal.Application/`, `Portal.Infrastructure/`, `Portal.API/`, `Portal.Web/`, `sql/`, `.github/`, `Directory.Build.props`, `nuget.config`, `.gitignore`, and `CLAUDE.md`.

### 3. Mark it as a Template Repository

This is the magic flag that adds the **Use this template** button.

Browser → `https://github.com/lfz-ai-powered-app/portal-template` → **Settings** (top-right gear) → scroll down to the **General** section → tick the checkbox **Template repository**.

Done. Refresh the repo page — a green **Use this template** button now appears next to the Code button.

### 4. (Optional) Let non-admins create from it

If your org restricts repo creation, go to:

**lfz-ai-powered-app org settings → Member privileges → Repository creation** → ensure "Allow members to create private repositories" is on.

Otherwise, builders will need you to click the button for them — which defeats the point.

---

## What lives in the template

Quick inventory so you know what you're shipping to builders:

| Path | Purpose |
|---|---|
| `Portal.Domain/` | Business-rules project (starts empty) |
| `Portal.Application/` | MediatR + FluentValidation scaffold |
| `Portal.Infrastructure/` | EF Core DbContext, ready for your additions |
| `Portal.API/` | ASP.NET Core 8 Web API with placeholder auth route |
| `Portal.Web/` | Blazor WebAssembly client with LFZ brand CSS |
| `sql/01-local-db.sql` | Idempotent creation script for IAM projection tables |
| `sql/02-seed-oneportal.sql.template` | OnePortal-side seed the builder fills in and sends you |
| `.github/workflows/ci.yml` | Build + test + secret-scan on every PR |
| `.github/PULL_REQUEST_TEMPLATE.md` | Mandatory checklist (cookies, CORS, validators, screenshots) |
| `CLAUDE.md` | Auto-loaded by Claude Code. Encodes the stack rules. |
| `Directory.Build.props` | Pins `OnePortalSdkVersion`, `TreatWarningsAsErrors` |
| `nuget.config` | Points at the LFZ private feed |
| `.gitignore` | Secrets, build output, IDE junk |

> **Note on OnePortal SDK wiring**: the template ships the NuGet references and a `CLAUDE.md` reminder, but the builder's big prompt (in `index.html` Step 4) explicitly tells Claude NOT to integrate OnePortal — it leaves a stub `/api/auth/me`. You replace the stub with real OnePortal wiring yourself after the PR lands. This is deliberate: builders stay focused on their feature; you own federation.

---

## Keeping the template fresh

Every time OnePortal changes in a way new portals need to pick up (new event type, SDK bump, brand refresh), do this in `portal-template`:

```bash
# In a checkout of lfz-ai-powered-app/portal-template
git checkout -b chore/bump-sdk
# ... make changes, e.g. bump OnePortalSdkVersion in Directory.Build.props ...
git commit -am "chore: bump OnePortal SDK to 1.2.*"
git push -u origin chore/bump-sdk
gh pr create --fill
```

Merge it. All *future* builders get the update automatically. Existing portal repos are NOT auto-updated — that's by design. If you need an existing portal to pick up a breaking change, open a PR against it manually or send the builder a targeted prompt.

**Recommended cadence**: review quarterly. Bump when OnePortal itself ships a breaking event contract, a new required route, or a brand palette change.

---

## Verifying it works (dry-run)

Before you send builders the link, try it yourself:

1. Go to `https://github.com/lfz-ai-powered-app/portal-template` → click **Use this template** → **Create a new repository**.
2. Name it `template-dryrun`, keep it private.
3. Create.
4. Clone the resulting repo in Visual Studio.
5. Press **Ctrl+F5**. You should see:
   - API Swagger at `https://localhost:7243`
   - Blazor app at `https://localhost:7154` with the LFZ navy header
   - No compile errors
6. Delete `template-dryrun` when satisfied.

If any of those fail, fix it in `portal-template` directly — don't work around it in individual builder repos.

---

## What to tell builders

Replace any mention of `degit` or manual scaffold steps with:

> "Go to **https://github.com/lfz-ai-powered-app/portal-template**, click **Use this template**, pick a name, and create the repo. Then clone YOUR new repo in Visual Studio."

The builder guide at `index.html` Step 2 has this flow spelt out with screenshots. Keep the guide's link somewhere they can find it (Teams / wiki / org README).

---

## Why not `degit`?

`degit` works, but:

- Requires Node.js installed (one more thing for non-tech users to get wrong).
- Ends up with no git history and no remote — builder has to `git init`, `gh repo create`, `git remote add`, `git push -u` in the right order. One slip and their first push fails confusingly.
- No "this was based on template X" trail.

"Use this template" solves all three silently.

---

## If you ever want to split brand/ out too

Some orgs end up with multiple brand variants (e.g. LFZ + a sister zone). Same pattern works: make a `portal-brand` repo, mark it as a template or publish its files as a NuGet package, and have the portal-template consume it. Not needed today — the one `portal-template` is fine for the foreseeable future.

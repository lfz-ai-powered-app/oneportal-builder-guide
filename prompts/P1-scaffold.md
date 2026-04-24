# P1 — Scaffold a New Portal App

**When to use:** right after `degit`ing the template, before anything else.

**What to replace:** every `<PORTAL_CODE>` with your portal code in CAPS (e.g. `ASSETTRACK`), every `<PortalName>` with the PascalCase name (e.g. `AssetTrack`).

---

## Paste this into Claude Code

```
Read the CLAUDE.md in the repo root and follow every rule in it as a hard constraint.

My portal code is <PORTAL_CODE> and the PascalCase name is <PortalName>.

Do the following in order:

1. Rename every `<Portal>` placeholder in the template tree to `<PortalName>`:
   - Folder names: `Portal.Domain` → `<PortalName>.Domain`, etc. for Application, Infrastructure, API, Web.
   - `.csproj` filenames.
   - `namespace` and `using` statements in every .cs file.
   - ProjectReference paths in every .csproj.
   - The root namespace in Directory.Build.props (if present).

2. Generate a solution file at the repo root called `<PortalName>.sln` referencing all five projects. Use `dotnet new sln` + `dotnet sln add`.

3. Run `dotnet restore` to verify project references resolve.

4. Pick a pair of unused ports for the API and Web. Defaults: API 7243 / 5081, Web 7154 / 5021. Update both `Properties/launchSettings.json` files.

5. In `<PortalName>.API/appsettings.json`, set `OnePortal:PortalCode` to "<PORTAL_CODE>". Leave `OnePortal:ServiceClient:ClientId` / `ClientSecret` empty — those go in User Secrets later.

6. Initialize a Git repo (`git init`), create `main` branch, first commit "chore: initial scaffold from oneportal-builder-guide template".

7. Create a blank EF migration named `InitialCreate`:
   `dotnet ef migrations add InitialCreate --project <PortalName>.Infrastructure --startup-project <PortalName>.API`

8. Print a summary:
   - The five project names and their purposes (one line each).
   - The port numbers you picked.
   - The next prompt to run (P2 — Set up local DB & seed).
   - Any files you could not rename and why.

Do NOT:
- Add any NuGet packages yet (that's P3's job).
- Write business logic.
- Change the OnePortal SDK-related scaffolding that's already in Program.cs — leave the placeholders until P3.
- Modify CLAUDE.md.
```

---

## What you should see

- A solution that builds (`dotnet build` succeeds) but does very little yet.
- One empty EF migration.
- A clean first commit on `main`.

## Verification

```bash
dotnet build
dotnet ef migrations list --project <PortalName>.Infrastructure --startup-project <PortalName>.API
git log --oneline
```

Expected: build clean, one migration named `InitialCreate`, one commit.

## Next

Run **P2 — Set Up Local DB & Seed**.

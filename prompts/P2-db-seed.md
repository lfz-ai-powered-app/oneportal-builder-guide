# P2 — Set Up Local DB & Generate OnePortal Seed

**When to use:** after P1, before P3.

**What it does:**

1. Configures your app to talk to your local SQL instance.
2. Runs the projection-table SQL (`01-local-db.sql`) against your DB.
3. Generates a per-portal OnePortal seed (`02-seed-oneportal.sql`) tailored to you.

The local DB script is for **your** machine. The OnePortal seed is for the **integrator** to run against the OnePortal dev DB — you never run it yourself.

---

## Before you paste

Know:

- **SQL instance**: either `(localdb)\MSSQLLocalDB` or `.\SQLEXPRESS` (check what you have in SSMS).
- **Database name**: `<PortalName>Db` (e.g. `AssetTrackDb`).
- **Your pilot email** — the account you'll use to test. The integrator will need to have created this user in OnePortal first.
- The list of **portal roles** you want. Default copies StoreTrack: `<PORTAL_CODE>_STAFF`, `<PORTAL_CODE>_DEPT_MANAGER`, `<PORTAL_CODE>_STORE_PERSON`, `<PORTAL_CODE>_STORE_MANAGER`. You can keep these or change them.

---

## Paste this into Claude Code

```
Read CLAUDE.md at the repo root first. Follow every rule.

Portal code: <PORTAL_CODE>
PortalName: <PortalName>
SQL instance: <SQL_INSTANCE>   (e.g. (localdb)\MSSQLLocalDB)
Local DB name: <PortalName>Db
Pilot email: <YOUR_EMAIL>
Portal roles (comma-separated):
  <PORTAL_CODE>_STAFF,
  <PORTAL_CODE>_DEPT_MANAGER,
  <PORTAL_CODE>_STORE_MANAGER

Do the following in order:

1. Update `<PortalName>.API/appsettings.Development.json`:
   - `ConnectionStrings:DefaultConnection` =
     "Server=<SQL_INSTANCE>;Database=<PortalName>Db;Trusted_Connection=True;TrustServerCertificate=True;Connection Timeout=60;Command Timeout=60;"
   - `OnePortal:Projection:EnableBootstrapFallback` = true
   - `OnePortal:Messaging:RequireBroker` = false  (in-memory messaging for dev)

2. Ensure `<PortalName>.API/appsettings.json` keeps `OnePortal:Projection:EnableBootstrapFallback` = false and `OnePortal:Messaging:RequireBroker` = true.

3. Store the OnePortal service-client secret in User Secrets:
   `dotnet user-secrets init --project <PortalName>.API`
   Prompt the user to paste their ClientId + ClientSecret, then:
   `dotnet user-secrets set "OnePortal:ServiceClient:ClientId" "<value>" --project <PortalName>.API`
   `dotnet user-secrets set "OnePortal:ServiceClient:ClientSecret" "<value>" --project <PortalName>.API`
   Never write these to appsettings.

4. Confirm `sql/01-local-db.sql` exists in the repo. If it doesn't, copy it from the template. DO NOT run it yet — print the command the user should run in SSMS:
   "Run sql/01-local-db.sql against <PortalName>Db in SSMS."

5. Generate `sql/02-seed-oneportal.sql` from `sql/02-seed-oneportal.sql.template` by substituting:
   - @PortalCode  → <PORTAL_CODE>
   - @PortalName  → <PortalName>
   - @PortalBaseUrl → https://localhost:7243  (the API URL)
   - @PortalRoles → the list provided above
   - @PilotEmail → <YOUR_EMAIL>
   The output must be idempotent: every INSERT guarded by IF NOT EXISTS or MERGE.

6. Apply EF migrations to the local DB:
   `dotnet ef database update --project <PortalName>.Infrastructure --startup-project <PortalName>.API`

7. Print a handoff summary:
   - The 01-local-db.sql command to run in SSMS.
   - The path to sql/02-seed-oneportal.sql (to send to the integrator).
   - What User Secrets values are now set.
   - The next prompt: P3 — Wire OnePortal Authentication.

Do NOT:
- Run 02-seed-oneportal.sql yourself.
- Put the ClientSecret in any committed file.
- Bypass User Secrets for convenience.
```

---

## Verification

```bash
dotnet user-secrets list --project <PortalName>.API
# Should show OnePortal:ServiceClient:ClientId and ClientSecret.

dotnet ef migrations list --project <PortalName>.Infrastructure --startup-project <PortalName>.API
# Should show InitialCreate applied.
```

Open SSMS → your DB → Tables → you should see `IamAccessProjections`, `ProcessedIamEvents`, `PortalRoleDefinitions`, `PortalRolePermissions` after you run `01-local-db.sql`.

## Next

Run **P3 — Wire OnePortal Authentication**.

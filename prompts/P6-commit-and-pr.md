# P6 — Commit & Open a PR

**When to use:** after P5 when you've finished a feature or fix and the security checklist in README §10 passes.

**What it does:** creates a feature branch, writes a conventional-commit message, pushes, and opens a PR against the central `lfz-ai-powered-app` organization using the PR template.

---

## Before you paste

Confirm:

- You're on a clean working tree (`git status` is clean) except for your intended changes.
- The remote `origin` exists. If not: `gh repo create lfz-ai-powered-app/<repo-name> --private --source=. --push`.
- You're signed in to GitHub CLI (`gh auth status`).
- `dotnet build` is clean and `dotnet test` is green.
- README §10 security checklist passes.

---

## Paste this into Claude Code

```
Read CLAUDE.md at the repo root first.

Follow this exact sequence.

1. Run `git status` and `git diff` to see what's changed. Summarize in 3 bullets what the change is about.

2. Classify the change type and pick a branch prefix:
   - `feat/` for a new feature
   - `fix/` for a bug fix
   - `chore/` for deps / tooling
   - `docs/` for docs only

3. Ask the user for a short branch slug (3-5 words, kebab-case). Default: derive it from the summary.

4. Create and check out the branch:
   `git checkout -b <prefix>/<slug>`

5. Stage the changes file-by-file (NOT `git add -A`). Skip any of these if they show up:
   - *.secrets.json
   - secrets.json
   - *.env, .env.*
   - *.pfx, *.key, *.pem
   - appsettings.*.local.json
   - Any file under `bin/` or `obj/`

6. Scan the staged diff for potential secrets:
   `git diff --cached | grep -iE "password|secret|apikey|api[_-]?key|bearer|connection[_-]?string"`
   If any match found, STOP and ask the user whether each is intentional. Never commit blind.

7. Write a conventional commit message with this structure:
   <type>(<scope>): <short imperative summary>

   <body — why the change exists, not what the diff shows>

   <footer — BREAKING CHANGE: ..., or Refs: #123>

   End with:
   Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>

8. Commit using HEREDOC:
   git commit -m "$(cat <<'EOF'
   <message>
   EOF
   )"

9. Push with upstream tracking:
   `git push -u origin <prefix>/<slug>`

10. Open a PR using the template at `.github/PULL_REQUEST_TEMPLATE.md`:
    gh pr create --title "<commit summary>" --body "$(cat <<'EOF'
    ## Summary
    <bullet the 3 most important things>

    ## Screenshots
    - [ ] App running standalone (https://localhost:<web-port>)
    - [ ] App running inside OnePortal iframe (https://localhost:7150)
    _(Attach both screenshots to the PR after creation.)_

    ## Seed SQL
    Attached: sql/02-seed-oneportal.sql
    - [ ] Re-runnable (every INSERT guarded with IF NOT EXISTS or MERGE)
    - [ ] Sent to the integrator for OnePortal DB seeding

    ## Security checklist (README §10)
    - [ ] Cookies HttpOnly + Secure + SameSite=None
    - [ ] No secrets committed
    - [ ] CORS pinned
    - [ ] [Authorize] on every endpoint or explicit [AllowAnonymous]
    - [ ] FluentValidation on every command/query
    - [ ] OnePortalUserId-only federated lookup
    - [ ] EnableBootstrapFallback: true only in Development

    ## Test plan
    - [ ] `dotnet build` clean
    - [ ] `dotnet test` green
    - [ ] Manual smoke test via OnePortal iframe
    - [ ] `/api/auth/me` returns 200 signed in, 401 signed out
    - [ ] Idempotency: ran 02-seed-oneportal.sql twice with no errors

    🤖 Generated with [Claude Code](https://claude.com/claude-code)
    EOF
    )"

11. Print the PR URL and remind the user to attach both screenshots.

Do NOT:
- Commit to main directly.
- Force-push to main.
- Skip hooks (--no-verify).
- Bypass GPG (--no-gpg-sign) unless the user explicitly asks.
- Use `git add -A` or `git add .`.
```

---

## Common commit message examples

```
feat(locations): add CRUD for warehouse locations

Implements the location-management feature requested by operations.
Uses the standard MediatR + FluentValidation pattern. Added the
manage-locations policy and wired it in Infrastructure/DI.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
```

```
fix(auth): remove legacy username fallback

Federated users must be looked up by OnePortalUserId only. The
legacy Username-matching branch also had an untranslatable LINQ
expression that broke all logins when EnableLegacyUserLinkFallback
was true. Removes the branch and its config flag.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
```

## After the PR is open

1. Attach both screenshots by editing the PR description.
2. Request the integrator as a reviewer.
3. Message the integrator with: PR link + path to `sql/02-seed-oneportal.sql`.
4. Watch CI. If red: read the logs, fix, push again (no rebase, no force).

# P4 — Apply LFZ Brand

**When to use:** after P3, before you build features.

**What it does:** ships your Blazor app with the same look and feel as OnePortal — Ubuntu font, LFZ navy + orange, Bootstrap Icons, Font Awesome, light/dark toggle.

---

## Paste this into Claude Code

```
Read CLAUDE.md at the repo root first. The rule about not mutating base CSS tokens is absolute.

PortalName: <PortalName>

Do the following in order:

1. Copy `brand/themes.css` and `brand/app.css` VERBATIM into `<PortalName>.Web/wwwroot/css/`. Do not edit them. If you need new tokens, add them to a new file `<PortalName>.Web/wwwroot/css/extensions.css` and reference that after the base files.

2. Copy `brand/assets/lfzlogo.png` into `<PortalName>.Web/wwwroot/images/`.

3. Update `<PortalName>.Web/wwwroot/index.html`:
   - Inside <head>, add (in this order):
     <link rel="preconnect" href="https://fonts.googleapis.com">
     <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
     <link href="https://fonts.googleapis.com/css2?family=Ubuntu:wght@400;500;700&family=Ubuntu+Mono&display=swap" rel="stylesheet">
     <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.3/font/bootstrap-icons.min.css">
     <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.0/css/all.min.css">
     <link rel="stylesheet" href="css/themes.css">
     <link rel="stylesheet" href="css/app.css">
     <link rel="stylesheet" href="css/extensions.css">
   - Replace the default <link rel="icon"> with the LFZ favicon.
   - Add the theme-toggle script block at the end of <body>:
     <script>
       window.lfzTheme = {
         current: () => document.documentElement.classList.contains('dark-theme') ? 'dark' : 'light',
         toggle: () => {
           const html = document.documentElement;
           html.classList.toggle('dark-theme');
           localStorage.setItem('lfz-theme', html.classList.contains('dark-theme') ? 'dark' : 'light');
         },
         apply: () => {
           if (localStorage.getItem('lfz-theme') === 'dark') document.documentElement.classList.add('dark-theme');
         }
       };
       window.lfzTheme.apply();
     </script>

4. Create `<PortalName>.Web/Shared/MainLayout.razor` matching OnePortal's chrome:
   - Top bar: background `#001f57` (portal navy), white text, LFZ logo on the left, user name + theme toggle button + logout on the right.
   - Left sidebar: `<NavMenu />` component, background uses `var(--bg-secondary)`.
   - Main content: uses `var(--bg-primary)` and `var(--text-primary)`.
   - Responsive: sidebar collapses below 768px.

5. Create `<PortalName>.Web/Shared/NavMenu.razor` with a placeholder nav list. Each item uses Bootstrap Icons (e.g. `<i class="bi bi-house"></i>`). Mark nav items with AuthorizeView based on permission claims.

6. Create `<PortalName>.Web/Shared/LfzSpinner.razor` — a loading spinner matching the one in OnePortal.Blazor (circular, LFZ orange). Use it as the `<RouteView DefaultLayout>` Loading fallback.

7. Verify the app still boots: `dotnet build`, then `dotnet run --project <PortalName>.Web`. Open https://localhost:7154 and confirm the top bar, sidebar, theme toggle all render correctly.

8. Print a summary:
   - Files created / modified.
   - Remind the user: DO NOT edit brand/themes.css. Add new tokens to extensions.css only.
   - Next prompt: P5 — Add a feature end-to-end.

Do NOT:
- Edit the base themes.css CSS variables.
- Hardcode colour hex values in Razor markup. Use var(--...) tokens.
- Load fonts or icons from any CDN other than the ones listed above.
- Add Tailwind CSS unless the user explicitly asks (the base stylesheet already handles utilities via custom props).
```

---

## Verification checklist

- [ ] `themes.css` byte-for-byte identical to `brand/themes.css`.
- [ ] Ubuntu font visible (letter shapes obviously different from default Segoe).
- [ ] Theme toggle flips `html` between light and dark, and the selection survives a page refresh.
- [ ] No hardcoded `#002b5c` etc. in any `.razor` file — search your repo to confirm.

## Next

Run **P5 — Add a Feature End-to-End**.

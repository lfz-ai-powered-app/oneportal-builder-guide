# LFZ Brand Bundle

This folder is the **single source of truth for OnePortal / Lagos Free Zone brand tokens**. It is copied verbatim from `OnePortal.Blazor/wwwroot/css/`. Do not hand-edit these files — when LFZ refreshes the palette, the integrator will republish this bundle and every portal picks it up.

## Contents

| File | What it is |
|---|---|
| `themes.css` | CSS custom properties for light + dark modes. Don't touch. |
| `app.css` | Global styles — Ubuntu font, scrollbars, toasts, spinner. Don't touch. |
| `assets/lfzlogo.png` | The official Lagos Free Zone logo. Don't resize or recolour. |

## Extending without editing the base

If your portal needs a custom token (e.g. a colour for a specific chart), create `<PortalName>.Web/wwwroot/css/extensions.css`:

```css
:root {
  --my-portal-chart-line: #0ea5e9;
}

html.dark-theme {
  --my-portal-chart-line: #7dd3fc;
}
```

Load it **after** `themes.css` in `index.html`. Never redefine the base tokens (`--bg-primary`, `--text-primary`, etc.).

## Palette cheat sheet

| Token | Value |
|---|---|
| LFZ deep blue | `#002b5c` |
| Portal navy | `#001f57` |
| LFZ accent orange | `#f04925` |
| Primary button | `#1e40af` (light) / `#3b82f6` (dark) |
| Success | `#10b981` |
| Error | `#ef4444` |
| Warning | `#f59e0b` |

Font: Ubuntu 400 / 500 / 700 from Google Fonts.
Icons: Bootstrap Icons 1.11.3 + Font Awesome 6.5.0 from CDN.

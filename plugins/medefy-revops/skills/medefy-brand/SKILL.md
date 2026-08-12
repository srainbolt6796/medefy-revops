---
name: medefy-brand
description: Medefy's official brand palette, logo, and styling. Reference when producing any branded deliverable (dashboards, slide decks, documents, reports) so colors, logo, and styling match Medefy.
---

# Medefy Brand

Official brand styling for Medefy deliverables. Apply to any branded output (HTML dashboards, PDFs, slide decks, documents).

## Brand color palette (from HubSpot brand settings)

| Hex | Role | Use for |
|---|---|---|
| `#2882FA` | Primary blue | Primary brand color: buttons, links, bar fills, key accents |
| `#1EAAFA` | Secondary blue | Secondary fills, gradients |
| `#1EC8F0` | Cyan | Tertiary accent, gradients, eyebrows, positive highlights |
| `#FF1EA0` | Magenta/Pink | High-impact accent: pink gradients, highlights, metadata text |
| `#202035` | Ink (dark navy) | Body text, headings, dark header backgrounds |
| `#E9EBEE` | Light grey | Page backgrounds, chart tracks, borders, dividers |
| `#FFFFFF` | White | Cards, surfaces |

**Balance blue with pink.** Use magenta/pink as a deliberate secondary accent so deliverables aren't all-blue. Pink text tint on dark: `#FF57B0`. Pink gradient for chips/links/buttons: `linear-gradient(135deg,#FF1EA0,#FF6FB5)`.

**Dark header banner (preferred for dashboards):** dark navy base with soft colored glows, white title, cyan eyebrow, pink subtitle.
```
background:
  radial-gradient(105% 150% at 6% 135%, rgba(30,200,240,.42), transparent 45%),
  radial-gradient(95% 150% at 102% -25%, rgba(150,80,240,.48), transparent 52%),
  radial-gradient(70% 130% at 84% 130%, rgba(255,30,160,.22), transparent 55%),
  linear-gradient(115deg,#1b1c3c 0%,#181934 55%,#211a3c 100%);
```

## Logo
The official **reversed logo** (colored chevron + light wordmark, for dark/colored backgrounds) is bundled with this skill as **`medefy-logo-reversed.png`**. To use it, read that file and base64-embed it into the HTML at build time (~40px tall). If the file isn't present in the session, ask the user to attach `med-logo-reversed-rgb.png`. Use a dark-wordmark logo on light backgrounds.

## Typography
- Brand font is **Galano Grotesque** (licensed) — use only where installed.
- Portable files: **Aptos** for Office docs; `'Inter','Segoe UI',system-ui,Arial,sans-serif` for HTML.

## Notes
- These hex values are the source of truth — keep in sync if Medefy updates brand settings.
- In HTML deliverables, expose the palette as CSS variables at the top of the file.

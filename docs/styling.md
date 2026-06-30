# Styling

Visual language and the canonical colour tokens. Status colours here are the source of truth; other docs
reference them by name.

## Theme

- **Light mode** — Figma/FigJam-inspired warm white palette.
- Typography: system font stack (`-apple-system, BlinkMacSystemFont, 'Segoe UI'`).
- Fully responsive — works on mobile browsers too.

## Surface & accent tokens

| Token | Value | Use |
|---|---|---|
| Background | `#F7F6F3` | warm off-white, like FigJam canvas |
| Card surface | `#FFFFFF` | cards, sticky feature column |
| Secondary surface | `#F2F0EB` | headers, muted fills |
| Border | `#E0DAD0` | dividers, cell borders |
| Accent | `#7B61FF` | Figma purple — links, primary buttons, login button |

## Status colours (also drive Timeline bars)

| Status | Colour | Notes |
|---|---|---|
| In Progress | `#7B61FF` purple | |
| Done | `#1BC47D` green | |
| On Hold | `#EAB308` **yellow** | changed from amber |
| Backlog | `#64748B` slate | Tasks page badge only |

**Done row tint:** completed features get a light-green wash on their Timeline Feature cell —
`.g-done-row .g-name { background: #E8F7EF; }` — alongside the green ✓, so done work reads at a glance.

## Priority colours

P1 `#E5484D` red · P2 `#F5A623` amber · P3 `#888076` grey.

## Calendar shading

- Weekend column: `rgba(60,50,35,0.16)`
- Holiday column: `rgba(245,166,35,0.30)`

(both darkened Jun 2026)

## Favicon

An inline **base64 SVG data URI** in `<head>` (no external request, so no `/favicon.ico` 404 and nothing
to host): a brand-purple (`#7B61FF`) rounded square with three white bars. Base64 (not raw-SVG) is used
because raw `<svg>` data URIs render inconsistently across browsers (Safari especially).

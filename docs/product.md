# Product

What the app is for and how each view behaves. For the data shape see
[data-model.md](./data-model.md); for how it's built see [architecture.md](./architecture.md).

## Overview

A single-page tool built for **daily personal use** to track design work across features, platforms,
and time. No backend logic — just a static page backed by `localStorage` and (optionally) Supabase
cloud sync. Built for Alan's design workflow at Jiraaf / altGraaf.

## Views

### Focus (nav tab "Focus"; view id stays `today`)
**Purpose:** Answer "what should I work on today and this week?" Three stacked horizons:

1. **⚠️ Overdue** callout (only when present) — `backlog` tasks whose `target` is before today,
   oldest-first, each with **Start working**. Red box.
2. **In progress** — a card per `in_progress` feature: name, tags, started date + days-ago + days-worked,
   the inline **"What did I do today?"** field (writes straight to `dayNotes[today]` on blur), and quick
   actions Edit / To Tasks / Mark Done.
3. **This week** — `backlog` tasks with `target` from today through end of this week (Mon–Sun).
4. **Upcoming** — `backlog` tasks with `target` after this week.

Tasks **without** a `target` stay on the Tasks page and don't appear in Focus. Week boundaries:
`startOfWeek` = Monday, `endOfWeek` = Sunday; an item due earlier this week but before today reads as overdue.

### Tasks / Backlog
**Purpose:** Capture things to do later, then promote them to the Timeline when you start.

- **Quick-add bar** — title (Enter to add), optional priority (P1/P2/P3), optional link, optional target
  date. Creates a `backlog` feature with no blocks. Links auto-prefix `https://`.
- Count badge + filters: search, priority, tag.
- **Task list** in manual order (drag to reorder — same flicker-free engine as the Timeline, persists to
  the shared `wt_features` array). Each row: drag handle, priority badge, name (click → edit), due badge
  (red if overdue), tag chips, clickable link chips, and **Start working** / **Edit** actions.
- Empty state when no tasks.

### Feature Log
**Purpose:** Full history of all features ever logged.

- Filter bar: status (All / In Progress / Done / On Hold), tag dropdown, year (from earliest block start),
  search by name.
- Feature list (newest first): name + tags, status badge, date range (first block start → last block
  end), **days worked** + daily-note count + PM, a compact status-coloured bar of each block, expand for
  the **Daily Log** (chronological per-day notes) + general notes, Edit / Delete.
- Empty state per filter combination.

### Timeline / Gantt (default landing)
**Purpose:** Visual calendar showing all features across time.

- Month/quarter toggle at the top (default: current month).
- **Left column** — each row stacks: brand badges, feature name (wraps), and a subtitle `Nd worked | PM:
  <name>` (PM omitted when unset). Tags collapse to small **brand badges** (+`+N` for custom tags);
  hover/focus reveals a popover with full chips.
- **Rows are NOT sorted by status** — they keep stored order (or manual reorder). **Drag-to-reorder** by
  grabbing a row's Feature cell: the row lifts and follows the cursor, others slide to open a gap, drop
  glows. Detection uses each row's midpoint captured at drag start (not live hit-testing) to avoid flicker.
- **Done indicator:** a green ✓ pins to the right of the Feature cell for `done` rows (yields to the drag
  handle ⠿ on hover), and the whole Feature cell gets a light-green tinge. See [styling.md](./styling.md).
- **In-progress features always get a row**, even with no block in view — so on the 1st of a month you can
  click a day and log work without opening the modal. Done/on-hold features appear only when a block overlaps.
- All block interactions use **Pointer Events** (works on touch; blocks set `touch-action: none`; empty
  cells still scroll; a tap opens the Day modal).
- **Add Feature affordance:** a thin bottom row, transparent at rest, revealing `+ Add feature` on hover.
- **Right canvas** — div grid, fixed **96px/day**, one row per feature:
  - Day headers: weekday abbrev + day number stacked ("Mon" / "1").
  - **One bar per work block**, coloured by status; gaps allowed; tall (~64px) to fit inline notes.
  - **Direct editing (FigJam-style):** click an empty day → 1-day block · click-drag across cells →
    multi-day block (modal opens range-prefilled) · drag edges to resize · drag body to move · click a
    block to log/view that day's note · Delete via the Day modal (confirm + undo).
  - **Weekends shaded.** A block dragged across a weekend splits; clicking a weekend cell logs that day.
  - Today column highlighted.
  - **Day notes render inline inside the bar** — a note "owns" subsequent days within its block until the
    next note/block end; clamped to 3 lines with full text on hover; don't carry across separate blocks.
  - **Notes follow the block when it moves** (`reanchorBlockNotes`): moving or left-edge resize shifts
    notes by the same delta; off-range/non-working notes snap to the first working day; collisions merge.
    Right-edge resize leaves the start put.
- A small **info (i) button** opens a "how this works" popover (replaces the always-visible help line).
- The **gantt area has its own scroll container** (max-height ≈ viewport − 200px). Month + day-header rows
  stick to top; the feature-name column stays sticky left across the full horizontal scroll
  (`.gantt-grid` uses `width: max-content`).
- Navigation: prev/next period arrows; a **Today** button jumps the anchor to the current period and
  horizontally scrolls so today's column is centred in view (`scrollTimelineToToday`). Filter by tag.

### Add / Edit Feature modal
**Fields:** Name (required) · Product Manager (free text) · Tags (multi-select + inline "Add new tag") ·
Status (segmented: **Backlog** / In Progress / Done / On Hold — Backlog hides the Work blocks section &
skips block validation) · Priority (None / P1 / P2 / P3) · Links (dynamic URL + optional label rows) ·
Target date (drives Focus's This week / Overdue) · Work blocks (dynamic start/end rows) · Notes (general
context, not the daily log). **Actions:** Save / Cancel / Delete (edit only).

**Name autocomplete & resume:** the Name field suggests every existing feature (a `<datalist>`). In
**Add** mode, if the typed name **exactly matches** an existing feature (case-insensitive), the modal
silently switches to *editing* that feature — reusing the same record rather than creating a duplicate,
so a paused/On-Hold feature can be picked back up later. A transient "Resuming …" toast confirms; no
persistent link UI is shown. (No effect when already editing a specific feature.)

**Validation:** Name required; at least one block with a start; a block's end can't precede its start.

### Day modal (from the Timeline, on clicking a day)
A **Working / Holiday** toggle:
- **Working** (default): a "what did you do?" log box + a **Feature status** segmented control (pre-set to
  current status) — so a block can be marked done in the same step. Saving stores the note, applies the
  status, and ensures a work block exists that day. If the day is inside an existing block, a **Delete**
  button removes the whole block (notes kept).
- **Holiday**: a reason field + From/To range. Saving marks every weekday in the range as a global holiday
  and re-splits all features' blocks around it. Deletes go through confirm + undo.

## Interactions & UX details

- **Navigation:** clicking nav items swaps the visible view, no reload.
- **Modals:** centered, capped at `max-height: 85vh` with internal scroll; only the Cancel/Save/Delete
  buttons close a modal — **Esc** and outside-click are disabled to avoid accidental data loss (Esc still
  closes the Timeline info popover).
- **Inline editing:** clicking a feature name anywhere opens the edit modal.
- **Persistence:** every save/delete writes to `localStorage` instantly (no explicit save button) and
  syncs to the cloud when configured.
- **Multi-tab:** a `storage` listener re-renders when another tab changes the data.
- **Dismiss keyboard (touch only):** on touch devices, focusing any text field shows a small floating
  "close keyboard" button (keyboard glyph + down chevron) pinned bottom-right (clearing the iOS safe
  area). Tapping it blurs the field so the on-screen keyboard slides away; it hides when no text field
  is focused. Never shown on desktop/mouse devices.
- **Export / Import (nav bar):** Export downloads `work-tracker-backup-YYYY-MM-DD.json` (features, tags,
  holidays, version, exportedAt). Import validates, confirms, then **replaces** all data (undo toast), and
  sets the one-time migration flags so seed/backfill routines don't re-run over imported data.

## Out of scope (this MVP)

- Year Review view (needs accumulated data first)
- Jira / external tool sync
- Multi-user collaboration (sync is single-user)
- Notifications or reminders
- Hour-level time tracking

## Implementation order (historical)

1. HTML skeleton — nav, view containers, modal shell
2. CSS — variables, reset, layout, typography, cards, modal
3. Data layer — localStorage read/write helpers, seed data
4. Today view — render, quick actions
5. Add/Edit modal — form, validation, save logic
6. Feature Log view — render list, filters, expand/collapse
7. Timeline/Gantt view — date grid, bar rendering, tooltip, navigation
8. Polish — empty states, transitions, responsive tweaks
9. Tasks backlog + Focus planning view
10. Cloud sync + hosting (see [changelog.md](./changelog.md))

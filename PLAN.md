# Work Tracker — HTML MVP Execution Plan

## Overview
A single `index.html` file with embedded CSS and JS. No backend. Data persists in `localStorage`. Built for daily personal use to track design work across features, platforms and stages.

---

## Data Model

### Feature
```json
{
  "id": "uuid",
  "name": "string (required)",
  "tags": ["string"],
  "status": "backlog | in_progress | done | on_hold",
  "pm": "string | null",
  "priority": "p1 | p2 | p3 | null",
  "links": [ { "url": "string", "label": "string" } ],
  "target": "YYYY-MM-DD | null",
  "blocks": [ { "id": "uuid", "start": "YYYY-MM-DD", "end": "YYYY-MM-DD" } ],
  "dayNotes": { "YYYY-MM-DD": "what I did that day" },
  "notes": "string | null",
  "createdAt": "ISO timestamp"
}
```

> **Tasks pipeline & Focus (Jun 2026).** A feature is one unit of work across its whole lifecycle: `backlog → in_progress → done` (plus `on_hold`). There is **one data store** (`wt_features`) and **one object shape** — a "task" is just a feature with `status: 'backlog'` and no blocks yet. Pages are filtered *views* of that store, not separate stores:
> - **Tasks page** shows `backlog` items only; **Timeline** shows `in_progress`/blocked items; **Log** excludes backlog; **Focus** is the planning home.
> - **Promote** ("Start working"): flips a backlog task to `in_progress`, drops a 1-day block at **today**, and it appears on the Timeline — the *same object* crossing a filter boundary (no copy/link).
> - **Send back to Tasks** reverts an item to `backlog` (always confirms; removes its blocks, keeps daily notes).
> - New optional fields are additive and back-compatible: `priority` (P1/P2/P3), `links` (URL + optional label, **no image embedding** by design — sidesteps localStorage size limits), `target` (a "do by" date that drives the Focus page's *This week* / *Overdue* lists). `migrateFeatures()` backfills `priority:null`, `links:[]`, `target:null` on load.

**Holidays** are stored separately (global, not per-feature) in `localStorage` under `wt_holidays`:
```json
{ "2026-01-26": "Republic Day", "2026-02-06": "Personal leave" }
```
Holidays behave like weekends: shaded (amber, vs weekend grey), excluded from days-worked, and blocks split around them (`normalizeBlocks` treats weekend OR holiday as a non-working day; a block made entirely of off-days is kept as intentional work).

> **Work blocks model (Jun 2026).** Stages (Ideation→Handoff) were dropped, then the single start/end date was replaced by **work blocks** — like the sticky blocks on Alan's FigJam board. A feature can have several blocks; gaps between them are days not worked. This makes the duration meaningful:
> - **Blocks don't span weekends — but weekend work can be logged explicitly.** A block that *spans* Sat/Sun is split into weekday runs — Thu→Tue becomes Thu–Fri + Mon–Tue (`normalizeBlocks`), so a dragged span never silently counts the weekend. **Exception:** a block made entirely of weekend days (you *clicked* a Sat/Sun) is kept intact — that's intentional weekend work. Weekends show as shaded columns either way. Split is applied as a one-time pass to existing data (`splitWeekendsOnce`, flag `wt_split_weekends_v1`) and on every drag/modal-save; it never strips a standalone weekend block.
> - **Days worked = distinct days across all blocks** (`featureWorkedDays`). Weekends only appear in blocks when explicitly added, so they only count when you meant them to.
> - On the Timeline you draw/edit blocks directly: **click any day (incl. weekends) to drop a block**, drag edges to resize, drag the body to move, hover + × to delete. Dragging a block across a weekend auto-splits it. Blocks are also editable as rows in the feature modal.
> - **Per-day notes** (`dayNotes`) are added by clicking a block (or via the Today card's inline field). Saving a note from the Today card also ensures a work block covers today (same as the Day modal), so the note always shows on the Timeline and counts as a worked day.
- **Working on a holiday never deletes the holiday.** Saving a Working day (single or dragged range) keeps the global holiday marked; a standalone block on the holiday is kept (like weekend work), and a range spanning a holiday splits around it.
> - `migrateFeatures()` is idempotent and auto-converts legacy records on load: stages → start/end → one block; an older single start/end → one block (ongoing features get an end of *today*). No data lost.
> - **Deletes are guarded:** every delete (feature, block, daily note, block-row) goes through a confirmation dialog, then an **undo toast** (6s). Reversible via a full-snapshot restore.
> - **`restoreSeedOnce()`** runs one time per browser (guarded by `wt_restored_v1`): it re-adds any seed feature missing *by name*, to recover an accidental wipe. It will not duplicate kept features and will not fight intentional deletes after that first run.

### Tags (stored separately in localStorage)
Default seed list:
- Jiraaf Web Desktop
- Jiraaf Web Mobile
- Jiraaf Mobile App
- altGraaf Web Desktop
- altGraaf Web Mobile
- altGraaf Mobile App

User can add custom tags at any time (e.g. `Design System`, `KYC`, `RA Insights`).

**Chip rendering:** wherever a tag is shown as a chip (Today cards, Log items, Timeline row labels, Modal tag picker), the brand prefix `Jiraaf`/`altGraaf` is replaced by the corresponding SVG from `Assets/Icons/` (`Jiraaf_Symbol.svg`, `altGraaf_Symbol.svg`). The platform suffix stays as text. Custom tags render as plain text. Stored tag strings are unchanged — only the rendering swaps in the icon. Native `<select>` dropdowns (log filter, timeline filter) keep showing the full text string since `<option>` can't render SVGs.

### Status (drives bar colour)
`backlog` (slate, Tasks page only) · `in_progress` (purple) · `done` (green) · `on_hold` (**yellow**, `#EAB308` — changed from amber)

### Priority (tasks)
`p1` High (red) · `p2` Medium (amber) · `p3` Low (grey) · `null` none. Shown as a small badge on Tasks rows and the Focus page.

---

## App Structure

```
index.html
  ├── <head>        CSS variables, reset, typography
  ├── <nav>         Top navigation (Timeline / Tasks / Focus / Log)
  ├── #timeline     Timeline / Gantt view (default landing)
  ├── #tasks        Tasks / Backlog view
  ├── #today        Focus view (Today + This week + Upcoming) — id is still "today"
  ├── #log          Feature Log view
  ├── #modal        Add / Edit feature modal (shared)
  └── <script>      All JS — data layer, rendering, interactions
```

One external dependency: the Supabase JS client (loaded from CDN) for optional cloud sync. The app still works as a pure local app when sync is unconfigured.

---

## Data Storage & Cloud Sync

**Local cache (always):** state lives in `localStorage` under `wt_features`, `wt_tags`, `wt_holidays`,
plus one-time migration flags (`ONE_TIME_FLAGS`). `load()`/`rawSave()` are the low-level accessors;
`save()` writes the cache synchronously **and** schedules a debounced cloud push.

**Cloud (Supabase, optional):** the source of truth when configured.
- **Config:** `SUPABASE_URL` + `SUPABASE_ANON` constants near the top of `<script>`. While they hold
  the `__PLACEHOLDER__` values, `SYNC_ENABLED` is false and the app behaves as before (no login, no
  network). The anon key is public by design; row-level security protects the data.
- **Schema:** one table `app_state(user_id uuid pk, data jsonb, updated_at timestamptz)`, RLS policy
  `auth.uid() = user_id`. The whole snapshot `{features, tags, holidays}` is stored as one JSONB row
  per user (same shape as the JSON export).
- **Auth:** email magic-link (`signInWithOtp`). `#auth-screen` overlay gates the app; session persists
  per device. A **Sign out** button sits in the nav.
- **Boot (`bootData`):** after auth, hydrate from the cloud row, then run the normal INIT
  (`seedIfEmpty → … → renderView`). Adopting cloud data sets `ONE_TIME_FLAGS` so seeds/backfills never
  re-run on authoritative data.
- **Push:** `save()` → `schedulePush()` (~1s debounce) → `pushNow()` upserts the snapshot.
  `#sync-status` shows `Saving… / Saved ✓ / Offline`.
- **Freshness:** Supabase Realtime subscription on the user's row + a pull on `window` focus; existing
  cross-tab `storage` listener retained. Conflict policy: last-write-wins by `updated_at` (single user).
- **No data loss:** `bootData` never blindly clobbers — cloud-null pushes local up; both-non-empty keeps
  the larger set; and `adoptSnapshot` stashes a `wt_local_backup_<ts>` copy before overwriting local
  (only when local actually differs from the incoming cloud snapshot, keeping just the last 5).
  The JSON export/import remains as a manual, fully-reversible backup path.

**Hosting:** deployed as a static page on GitHub Pages (`alanjmoses/work-tracker` →
`https://alanjmoses.github.io/work-tracker/`), which is the redirect URL allow-listed in Supabase Auth.

---

## Views

### 1. Focus View (upgraded "Today" — Phase 2)
**Purpose:** Answer "what should I work on today and this week?" The nav tab is labelled **Focus** (the view id stays `today`). Three stacked horizons:

1. **⚠️ Overdue** callout (only rendered when present) — `backlog` tasks whose `target` is before today, sorted oldest-first, each with a **Start working** button. Red box.
2. **In progress** — a card per `in_progress` feature: name, tags, started date + days-ago + days-worked, the **inline "What did I do today?"** field (writes straight to `dayNotes[today]` on blur), and quick actions Edit / To Tasks / Mark Done.
3. **This week** — `backlog` tasks with `target` from today through the end of this week (Mon–Sun), with the week range shown. Each row: priority badge, name, due badge, Start working.
4. **Upcoming** — `backlog` tasks with `target` after this week.

Tasks **without** a `target` stay on the Tasks page and don't appear in Focus — only scheduled work surfaces here. Week boundaries: `startOfWeek` = Monday, `endOfWeek` = Sunday; an item due earlier this week but before today reads as **overdue**.

---

### 1b. Tasks / Backlog View (Phase 1)
**Purpose:** Capture things to do later, then promote them to the Timeline when you start.

**Contents:**
- **Quick-add bar** — title (Enter to add), optional priority (P1/P2/P3), optional link, optional target date. Creates a `backlog` feature with no blocks. Links auto-prefix `https://`.
- Count badge + filters: search, priority, tag.
- **Task list** in manual order (drag to reorder — same flicker-free engine as the Timeline, persists to the shared `wt_features` array). Each row: drag handle, priority badge, name (click → edit), due badge (red if overdue), tag chips, clickable link chips, and **Start working** / **Edit** actions.
- Empty state when no tasks.

---

### 2. Feature Log View
**Purpose:** Full history of all features ever logged.

**Contents:**
- Filter bar:
  - By status: All / In Progress / Done / On Hold
  - By tag: dropdown of all tags
  - By year: derived from earliest block start
  - Search by name
- Feature list (newest first by default):
  - Feature name + tags
  - Status badge
  - Date range: first block start → last block end
  - **Days worked** (distinct days across blocks) + count of daily notes + PM when set
  - Compact status-coloured bar showing each block as a segment
  - Expand to see the **Daily Log** (all per-day notes, chronological) + general notes
  - Edit / Delete actions
- Empty state per filter combination

---

### 3. Timeline / Gantt View (default landing)
**Purpose:** Visual calendar showing all features across time.

**Contents:**
- Month/quarter toggle at the top (default: current month)
- Left column: each row stacks three blocks: (1) brand badges on their own line, (2) feature name (wraps to multiple lines if long), (3) a single subtitle line `Nd worked | PM: <name>` (PM segment omitted when not set). Tags are collapsed to small **brand badges** (one Jiraaf or altGraaf SVG per unique brand) plus a `+N` badge for any custom tags. Hovering (or keyboard-focusing) the badge cluster reveals a popover with the full chip list. Today / Log views still show full chips.
- **Rows are NOT sorted by status.** They keep their stored order (order added, or as manually reordered) — marking a feature Done no longer moves it. **Drag-to-reorder** by grabbing a row's Feature cell: the row lifts and follows the cursor, the other rows slide to open a gap (iOS-home-screen style), and on drop the moved row briefly glows. Detection uses each row's midpoint captured at drag start (not live hit-testing) to avoid flicker. The new order persists to `wt_features`.
- **Done check-mark:** a green ✓ pins to the right of the Feature cell for `done` rows; it yields to the drag handle (⠿) on hover.
- **In-progress features always get a row**, even when they have no block in the visible period — so on the 1st of a new month you can click a day and log work without opening the edit modal. Done/on-hold features only appear when a block overlaps the period.
- All block interactions use **Pointer Events**, so moving/resizing blocks also works on touch devices (blocks set `touch-action: none`; empty cells still allow touch-scrolling, and a tap on a cell opens the Day modal).
- **Add Feature affordance:** a thin row at the bottom of the gantt table that's transparent at rest and reveals a `+ Add feature` bar on hover. Click it to open the Add Feature modal. (Replaces the old floating + button — Today/Log views no longer have an inline add affordance; switch to Timeline to add.)
- Right: horizontal canvas (div grid, fixed **96px/day**) — each feature gets one row
  - Day headers show **weekday abbrev + day number** stacked ("Mon" / "1").
  - **One bar per work block**, coloured by status. A feature can show several blocks with gaps. Bars are tall (~64px) to fit inline note text.
  - **Direct editing (FigJam-style):** click an empty day to drop a 1-day block · **click-and-drag across cells to create a multi-day block** (the modal opens with the range pre-filled, save commits it) · drag a block's edges to resize · drag its body to move · click a block to log/view that day's note. To remove a block, click it and pick **Delete** in the Day modal (confirm + undo).
  - **Weekends are shown as shaded columns.** A block dragged across a weekend splits (Thu–Fri + Mon–Tue) so the weekend isn't counted — but clicking a weekend cell logs work for that day (a standalone weekend block that is kept and counted).
  - Today column highlighted.
  - **Day notes render inline inside the bar.** A note "owns" all subsequent days within the same block until the next note (or the block end). Long notes are clamped to 3 lines with full text on hover. Notes don't carry across separate blocks — each block starts blank. (Replaces the old "note dot" indicator.)
  - **Notes follow the block when it moves.** Moving a block or dragging its **left** edge re-anchors its notes (`reanchorBlockNotes`): they shift by the same amount the block's start moved, so the leading note lands on the new **first section**. Notes that would land on a non-working day or off the new range snap to the first working day; collisions merge (newline-joined). Right-edge resize leaves the start put, so it's untouched.
- A small **info (i) button** near the filter/toggle opens a popover with the "how this works" hint (replaces the always-visible help line).
- The **gantt area has its own scroll container** (max-height ≈ viewport − 200px). The month + day-header rows stick to the top while scrolling vertically; the feature-name column stays sticky on the left and is now **persistent across the full horizontal scroll** (`.gantt-grid` uses `width: max-content` so each row spans the whole track and the sticky column no longer detaches past one screen-width).
- Navigation: previous / next period arrows
- Filter: by tag (to reduce noise)
- Bar colour by status: In Progress — purple · Done — green · On Hold — amber

---

### 4. Add / Edit Feature Modal
**Purpose:** Create a new feature or edit an existing one.

**Fields:**
- Name (text input, required)
- Product Manager (text input, optional — free text, surfaces on Timeline and Log)
- Tags (multi-select from tag list + inline "Add new tag" option)
- Status (segmented control: **Backlog** / In Progress / Done / On Hold). Choosing **Backlog hides the Work blocks section** — backlog items have no blocks, and the block validation is skipped for them.
- Priority (segmented control: None / P1 / P2 / P3)
- Links (dynamic list of URL + optional label rows; + Add link / remove)
- Target date (optional — drives the Focus page's This week / Overdue lists)
- Work blocks: a dynamic list of start/end rows (+ Add block / remove). Same blocks you can draw on the Timeline.
- Notes (textarea, optional — general context, not the daily log)
- Actions: Save / Cancel / Delete (on edit only)

A **Day modal** opens from the Timeline whenever you click a day (cell or block). It has a **Working / Holiday** toggle:
- **Working** (default): a "what did you do?" log box plus a **Feature status** segmented control (In Progress / Done / On Hold, pre-set to the feature's current status) — so a block drawn on the Timeline can be marked done in the same step. Saving stores the note, applies the status, and ensures a work block exists on that day. If the clicked day is inside an existing block, a **Delete** button removes the entire block (notes kept) — replacing the old hover-× on the bar.
- **Holiday**: a reason field + a From/To range. Saving marks every weekday in the range as a global holiday and re-splits all features' blocks around the new holidays. Deletes go through confirm + undo toast like everything else.

**Validation:**
- Name required
- At least one block, each with a start; a block's end can't be before its start

---

## Interactions & UX Details

- **Navigation:** Clicking nav items swaps visible view, no page reload
- **Modals:** centered on the viewport, capped at `max-height: 85vh` with internal scroll if content overflows. Press **Esc** to close any open modal (also closes the Timeline info popover).
- **Modal:** Opens over current view, closes on Save / Cancel / outside click
- **Inline editing:** Clicking a feature name anywhere opens the edit modal
- **Mark as Done:** Quick action on Today cards — sets status to `done`
- **Delete:** Confirmation prompt before removing a feature
- **Tag management:** Adding a new tag in the modal saves it to the global tag list immediately
- **Persistence:** Every save/delete writes to `localStorage` instantly — no explicit save button for the app itself
- **Multi-tab:** a `storage` listener re-renders when another tab changes the data
- **Export / Import (nav bar):** Export downloads a JSON backup (`work-tracker-backup-YYYY-MM-DD.json` with features, tags, holidays, version, exportedAt). Import reads a backup file, validates it, confirms, then **replaces** all data (undo toast offered). Import also sets the one-time migration flags so seed/backfill routines don't re-run over imported data.

---

## Colour & Style

- **Light mode** — Figma/FigJam-inspired warm white palette
- Background: `#F7F6F3` (warm off-white, like FigJam canvas)
- Card surfaces: `#FFFFFF`
- Secondary surface: `#F2F0EB`
- Borders: `#E0DAD0`
- Accent: `#7B61FF` (Figma purple)
- Typography: system font stack (`-apple-system, BlinkMacSystemFont, 'Segoe UI'`)
- Status colours (also drive the Timeline bars):
  - In Progress — `#7B61FF` purple
  - Done — `#1BC47D` green
  - On Hold — `#EAB308` **yellow** (changed from amber)
  - Backlog — `#64748B` slate (Tasks page badge only)
- Priority colours: P1 `#E5484D` red · P2 `#F5A623` amber · P3 `#888076` grey
- Weekend column shading: `rgba(60,50,35,0.16)` · Holiday shading `rgba(245,166,35,0.30)` (both darkened Jun 2026)
- Fully responsive — works on mobile browser too

## Seed Data

Full historical data seeded from **October 2025 → June 2026** based on Alan's FigJam roadmap board. Covers:
- Oct 2025: Jiraaf App FD Details, Jiraaf Web FD Listing, Bond Analyser Revamp, RA Insights (start)
- Nov 2025: altGraaf FD Listing & Details, Unit Selection Improvements, Pending Orders
- Dec 2025: RA Insights (hi-fi), Opportunity Details Redesign
- Jan 2026: RA Insights Details Page, IFA Investment Activity, KYC Flow DOB Addition
- Feb 2026: Header Revamp, Form 15G/H, Profile Edit
- Mar 2026: HUF KYC Flow, Income Certificate, Form 121, Figma File Restructuring
- Apr 2026: FTI Homepage Redesign, Disbursal Flow Admin, Trust Markers Phase 1, Design System Structuring
- May 2026: Accessibility Audit, Trust Markers Phase 2, Audit Requirements FATCA
- Jun 2026: KYC Not-Done Users Homepage, Opportunity Details Redesign v2

---

## Implementation Order

1. HTML skeleton — nav, view containers, modal shell
2. CSS — variables, reset, layout, typography, cards, modal
3. Data layer — localStorage read/write helpers, seed data
4. Today view — render, quick actions
5. Add/Edit modal — form, validation, save logic
6. Feature Log view — render list, filters, expand/collapse
7. Timeline/Gantt view — date grid, bar rendering, tooltip, navigation
8. Polish — empty states, transitions, responsive tweaks

---

## FigJam Import (one-time)

`importFigjamOnce()` (flag `wt_figjam_import_v1`) seeds Alan's real history transcribed from the FigJam roadmap (Oct 2025 → Jun 2026):
- **~106 daily notes** mapped to features by name, on the first day of each task span (overwrites existing notes on those dates).
- **7 new features** for stickies that had no matching feature: Create Tab view for Sell, Location required messaging, App API Loading error states, APD Interviews (hiring), Internal Presentation, Automating Internal Presentation Design, Bond Baskets.
- **10 high-confidence holidays** (public holidays + clearly-marked single days off). Ambiguous light-grey stretches were left out — Alan marks those via the Day modal's Holiday tab.
Notes were transcribed from screenshots (best-effort); a few tiny sub-labels may need correction in-app.

## PM per Feature (from FigJam workstreams)

Each feature's PM, inferred from the workstream it sat under in the FigJam roadmap. PMs: Ganesh, Pranchal, Vaishali, Rishu. **Wired into the data model** as `feature.pm` (string | null); `backfillPMsOnce()` (flag `wt_pm_backfill_v1`) seeds this list onto existing features by name on first load. Editable from the Add/Edit Feature modal.

| Feature | PM |
|---|---|
| Jiraaf App FD Details | Ganesh |
| Jiraaf Web FD Listing | Ganesh |
| altGraaf FD Listing & Details | Ganesh |
| Bond Analyser Revamp | Ganesh |
| RA Insights | Ganesh |
| RA Insights Details Page | Ganesh |
| Opportunity Details Redesign | Ganesh |
| Unit Selection Improvements | Pranchal |
| Pending Orders | Pranchal |
| IFA Investment Activity | — (not labelled — Jan, Workstream 2) |
| KYC Flow — DOB Addition | — (not labelled — Jan, Workstream 2) |
| Header Revamp | Ganesh |
| Form 15G/H — Multiple Issuer Email | Pranchal |
| Profile Edit — Mobile & Email | Ganesh |
| HUF KYC Flow | Ganesh |
| Income Certificate | Pranchal |
| Form 121 | Pranchal |
| Figma File Restructuring | — (Improving workflow — no PM) |
| FTI Homepage Redesign | Vaishali |
| Disbursal Flow Admin | Ganesh |
| Design System Structuring | — (Improving workflow — no PM) |
| Trust Markers Phase 1 | Vaishali |
| Trust Markers Phase 2 | Vaishali |
| Accessibility Audit | Vaishali |
| Audit Requirements — FATCA | Rishu |
| KYC Not-Done Users Homepage | Vaishali |
| Opportunity Details Redesign v2 | Ganesh |
| Create Tab view for Sell | Ganesh |
| Location required messaging | Pranchal |
| App API Loading error states | Pranchal |
| APD Interviews (hiring) | — (hiring, N/A) |
| Internal Presentation | — (internal) |
| Automating Internal Presentation Design | — (internal) |
| Bond Baskets | Ganesh |

## Out of Scope (this MVP)

- Year Review view (needs accumulated data first)
- Jira / external tool sync
- Multi-user / auth
- Notifications or reminders
- Hour-level time tracking

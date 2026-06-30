# Data Model

The single source of truth is `wt_features` (synced to Supabase — see
[architecture.md](./architecture.md)). For seeded historical content and the PM table, see
[seed-data.md](./seed-data.md).

## Feature

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

> **Tasks pipeline & Focus (Jun 2026).** A feature is one unit of work across its whole lifecycle:
> `backlog → in_progress → done` (plus `on_hold`). There is **one data store** (`wt_features`) and
> **one object shape** — a "task" is just a feature with `status: 'backlog'` and no blocks yet. Pages
> are filtered *views* of that store, not separate stores:
> - **Tasks page** shows `backlog` items only; **Timeline** shows `in_progress`/blocked items; **Log**
>   excludes backlog; **Focus** is the planning home.
> - **Promote** ("Start working"): flips a backlog task to `in_progress`, drops a 1-day block at
>   **today**, and it appears on the Timeline — the *same object* crossing a filter boundary (no copy/link).
> - **Send back to Tasks** reverts an item to `backlog` (always confirms; removes its blocks, keeps daily notes).
> - New optional fields are additive and back-compatible: `priority` (P1/P2/P3), `links` (URL + optional
>   label, **no image embedding** by design — sidesteps localStorage size limits), `target` (a "do by"
>   date that drives the Focus page's *This week* / *Overdue* lists). `migrateFeatures()` backfills
>   `priority:null`, `links:[]`, `target:null` on load.

## Holidays

Stored separately (global, not per-feature) in `localStorage` under `wt_holidays`:

```json
{ "2026-01-26": "Republic Day", "2026-02-06": "Personal leave" }
```

Holidays behave like weekends: shaded (amber, vs weekend grey), excluded from days-worked, and blocks
split around them (`normalizeBlocks` treats weekend OR holiday as a non-working day; a block made
entirely of off-days is kept as intentional work).

## Work blocks model (Jun 2026)

Stages (Ideation→Handoff) were dropped, then the single start/end date was replaced by **work blocks**
— like the sticky blocks on Alan's FigJam board. A feature can have several blocks; gaps between them
are days not worked. This makes the duration meaningful:

- **Blocks don't span weekends — but weekend work can be logged explicitly.** A block that *spans*
  Sat/Sun is split into weekday runs — Thu→Tue becomes Thu–Fri + Mon–Tue (`normalizeBlocks`), so a
  dragged span never silently counts the weekend. **Exception:** a block made entirely of weekend days
  (you *clicked* a Sat/Sun) is kept intact — intentional weekend work. Weekends show as shaded columns
  either way. Split is applied as a one-time pass to existing data (`splitWeekendsOnce`, flag
  `wt_split_weekends_v1`) and on every drag/modal-save; it never strips a standalone weekend block.
- **Days worked = distinct days across all blocks** (`featureWorkedDays`). Weekends only appear in
  blocks when explicitly added, so they only count when you meant them to.
- On the Timeline you draw/edit blocks directly: **click any day (incl. weekends) to drop a block**,
  drag edges to resize, drag the body to move, hover + × to delete. Dragging a block across a weekend
  auto-splits it. Blocks are also editable as rows in the feature modal.
- **Per-day notes** (`dayNotes`) are added by clicking a block (or via the Today card's inline field).
  Saving a note from the Today card also ensures a work block covers today, so the note always shows on
  the Timeline and counts as a worked day.
- **Working on a holiday never deletes the holiday.** A standalone block on the holiday is kept (like
  weekend work); a range spanning a holiday splits around it.
- `migrateFeatures()` is idempotent and auto-converts legacy records on load: stages → start/end → one
  block; an older single start/end → one block (ongoing features get an end of *today*). No data lost.
- **Deletes are guarded:** every delete (feature, block, daily note, block-row) goes through a
  confirmation dialog, then an **undo toast** (6s). Reversible via a full-snapshot restore.
- **`restoreSeedOnce()`** runs one time per browser (guarded by `wt_restored_v1`): re-adds any seed
  feature missing *by name*, to recover an accidental wipe. It won't duplicate kept features and won't
  fight intentional deletes after that first run.

## Tags (stored separately in localStorage)

Default seed list: Jiraaf Web Desktop, Jiraaf Web Mobile, Jiraaf Mobile App, altGraaf Web Desktop,
altGraaf Web Mobile, altGraaf Mobile App. User can add custom tags at any time (e.g. `Design System`,
`KYC`, `RA Insights`).

**Chip rendering:** wherever a tag is shown as a chip (Today cards, Log items, Timeline row labels,
Modal tag picker), the brand prefix `Jiraaf`/`altGraaf` is replaced by the corresponding SVG from
`Assets/Icons/` (`Jiraaf_Symbol.svg`, `altGraaf_Symbol.svg`). The platform suffix stays as text.
Custom tags render as plain text. Stored tag strings are unchanged — only the rendering swaps in the
icon. Native `<select>` dropdowns (log filter, timeline filter) keep showing the full text string since
`<option>` can't render SVGs.

## Status & priority (semantics)

- **Status** (drives Timeline bar colour): `backlog` (Tasks page only) · `in_progress` · `done` ·
  `on_hold`. Colour values live in [styling.md](./styling.md).
- **Priority** (tasks): `p1` High · `p2` Medium · `p3` Low · `null` none. Shown as a small badge on
  Tasks rows and the Focus page.

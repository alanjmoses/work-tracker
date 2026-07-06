# Changelog

Dated record of notable changes. Most recent first.

## 2026-07-06 — Modals no longer dismiss on Esc / outside-click

- Removed the outside-click handlers on all three modal overlays (Add/Edit Feature, Day, Confirm) and
  removed the modal-closing branch of the global Escape handler. Modals now only close via their
  Cancel/Save/Delete buttons — prevents accidental loss of in-progress edits from a stray click or Esc.
  Esc still closes the Timeline info popover.

## 2026-07-01 — Resume features by name + jump-to-today

- **Add Feature name autocomplete:** the name field is now backed by a `<datalist>` of every existing
  feature name, so typing suggests past features.
- **Exact match resumes the feature:** in Add mode, typing (or picking) a name that exactly matches an
  existing feature (case-insensitive) switches the modal to *editing* that feature instead of creating a
  duplicate — so you can pause a feature (e.g. On Hold) and pick it up later against the same record. A
  transient toast confirms ("Resuming …"); no persistent link UI is shown.
- **Timeline "Today" scrolls to today:** clicking **Today** now jumps the anchor to the current period
  *and* horizontally scrolls the gantt so today's column is centred in the visible track
  (`scrollTimelineToToday`).
- **Dismiss-keyboard button (touch only):** focusing a text field on a touch device shows a small
  floating keyboard-with-down-chevron button (bottom-right, safe-area aware); tapping it blurs the field
  to close the on-screen keyboard. Hidden on desktop/mouse devices.

## 2026-06-30 — Cloud sync, hosting, and polish

Moved the app from browser-only `localStorage` to hosted, synced storage, then ironed out the rough
edges. Full design in [architecture.md](./architecture.md).

### Cloud sync + auth (Supabase)
- Data now syncs to **Supabase** (source of truth) with `localStorage` as a local cache. Added the
  Supabase JS client (CDN), config constants (`SUPABASE_URL` / `SUPABASE_ANON`), and a `SYNC_ENABLED`
  flag that keeps the app a pure local app until configured.
- **Email magic-link login** (`signInWithOtp`) via an `#auth-screen` gate, plus a **Sign out** button and
  a `#sync-status` indicator (`Saving… / Saved ✓ / Offline`).
- Table `app_state(user_id, data jsonb, updated_at)` with **row-level security** (`auth.uid() = user_id`).
- `save()` wrapped to push (debounced ~1s); `bootData()` hydrates from cloud before the normal INIT;
  Realtime subscription + focus-pull keep devices fresh.

### Hosting
- Deployed to **GitHub Pages** (public repo `alanjmoses/work-tracker` → `alanjmoses.github.io/work-tracker/`).
- Hit (and documented) GitHub Pages' single-deploy-at-a-time concurrency: rapid back-to-back pushes
  cancelled/failed an in-flight deploy; a single clean redeploy fixed it.

### Data-safety fixes
- **No-op backups removed:** `adoptSnapshot` was writing a `wt_local_backup_<ts>` on every cloud adopt
  (i.e. every refresh on a synced device), accumulating unbounded. Now it only backs up when local
  genuinely differs and **caps at the 5 most recent**.
- **jsonb key-order bug:** the "did local change?" check used `JSON.stringify`, but Supabase `jsonb`
  returns keys in a different order, so it always reported a difference → backup every refresh. Fixed with
  `stableStringify` (sorted-key canonical compare).
- **Quieter UX:** routine refreshes are now silent; the "Loaded from cloud — local backup saved" toast
  only fires on a real divergence (`adoptSnapshot` reports whether it actually backed up).

### Polish
- **Favicon:** added an inline base64 SVG (brand-purple bar chart) — silences the `/favicon.ico` 404 and
  needs no hosted file.
- **Done-row tint:** completed features get a light-green wash (`#E8F7EF`) on their Timeline Feature cell,
  alongside the existing green ✓.

---

> Earlier history (Phases 1–2: Timeline/Gantt, Tasks backlog, Focus view, work-blocks model, FigJam
> import) predates this changelog and lives in the git log and the per-topic docs.

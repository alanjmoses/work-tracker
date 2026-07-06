# Architecture

How the app is put together and where data lives. For the data *shape*, see
[data-model.md](./data-model.md); for visuals, [styling.md](./styling.md).

## App structure

A single `index.html` file with embedded CSS and JS.

```
index.html
  ├── <head>        CSS variables, reset, typography, inline favicon, Supabase client
  ├── <nav>         Top navigation (Timeline / Tasks / Focus / Log) + Export/Import/Sign out
  ├── #auth-screen  Cloud-sync login gate (shown only when sync is configured and no session)
  ├── #timeline     Timeline / Gantt view (default landing)
  ├── #tasks        Tasks / Backlog view
  ├── #today        Focus view (Today + This week + Upcoming) — id is still "today"
  ├── #log          Feature Log view
  ├── #modal        Add / Edit feature modal (shared)
  └── <script>      All JS — data layer, cloud sync, rendering, interactions
```

One external dependency: the Supabase JS client (loaded from CDN) for optional cloud sync.
The app still works as a pure local app when sync is unconfigured.

## Data storage & cloud sync

**Local cache (always):** state lives in `localStorage` under `wt_features`, `wt_tags`,
`wt_holidays`, plus one-time migration flags (`ONE_TIME_FLAGS`) and `wt_local_edited_at` (stamped on
every local mutation — see below). `load()`/`rawSave()` are the low-level accessors; `save()` writes
the cache synchronously, stamps `wt_local_edited_at`, **and** schedules a debounced cloud push.

**Cloud (Supabase, optional):** the source of truth when configured.

- **Config:** `SUPABASE_URL` + `SUPABASE_ANON` constants near the top of `<script>`. While they hold
  the `__PLACEHOLDER__` values, `SYNC_ENABLED` is false and the app behaves as a pure local app (no
  login, no network). The anon key is public by design; row-level security protects the data.
- **Schema:** one table `app_state(user_id uuid pk, data jsonb, updated_at timestamptz)`, RLS policy
  `auth.uid() = user_id`. The whole snapshot `{features, tags, holidays}` is stored as one JSONB row
  per user (same shape as the JSON export).
- **Auth:** email magic-link (`signInWithOtp`). `#auth-screen` overlay gates the app; session persists
  per device. A **Sign out** button sits in the nav.
- **Boot (`bootData`):** after auth, hydrate from the cloud row, then run the normal INIT
  (`seedIfEmpty → migrateFeatures → … → renderView`). Adopting cloud data sets `ONE_TIME_FLAGS` so
  seeds/backfills never re-run on authoritative data (mirrors the import path).
- **Push:** `save()` → `schedulePush()` (~1s debounce) → `pushNow()` upserts the snapshot.
  `#sync-status` shows `Saving… / Saved ✓ / Offline`. A pending push is also flushed immediately on
  `visibilitychange` (tab hidden) / `pagehide`, to shrink the reload race described below.
- **Freshness:** Supabase Realtime subscription on the user's row + a pull on `window` focus; existing
  cross-tab `storage` listener retained. Conflict policy: **most-recently-edited wins**, compared by
  timestamp (see below) — not last-write-wins-by-arrival, since single-user multi-device edits can race.

### No-data-loss guarantees

- `bootData` never blindly clobbers, and the winner is decided by **recency, not feature count**:
  **cloud-null** pushes local up; **fresh device** (local empty) adopts cloud; **both non-empty** compares
  `wt_local_edited_at` (stamped by every `save()`) against the cloud row's `updated_at` — whichever is
  newer wins. This matters because adding/removing a block inside an *existing* feature doesn't change
  the feature count, so a count-based check would wrongly call it a tie; a reload racing the 1s debounced
  push would then silently discard the unsynced edit. Timestamp comparison keeps local intact in that case
  and re-pushes it.
- `adoptSnapshot` stashes a `wt_local_backup_<ts>` copy before overwriting local — but **only when local
  genuinely differs** from the incoming cloud snapshot, compared with `stableStringify` (key-order-insensitive,
  because Supabase `jsonb` reorders keys). Capped at the **5** most recent. Adopting cloud also stamps
  `wt_local_edited_at` to the cloud row's `updated_at`, so a later boot doesn't mistake "just adopted cloud"
  for "has newer unsynced local edits".
- Routine refreshes are **silent**; the "Loaded from cloud" toast only fires when a real divergence was
  backed up (`adoptSnapshot` returns whether it wrote a backup).
- The JSON export/import remains a manual, fully-reversible backup path.

## Hosting / deployment

Deployed as a static page on **GitHub Pages**: repo `alanjmoses/work-tracker` (public) →
`https://alanjmoses.github.io/work-tracker/`, which is the redirect URL allow-listed in Supabase Auth.

- Deploy = push to `main`; GitHub Pages' "pages build and deployment" workflow publishes the root.
- Pages allows only one deploy at a time — pushing several commits back-to-back can cancel/fail the
  in-flight deploy. Space out pushes or trigger a single clean redeploy if one fails.
- The favicon is an inline base64 SVG data URI (no external request); see [styling.md](./styling.md).

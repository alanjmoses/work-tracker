# Work Tracker

A single `index.html` app for tracking design work across features, platforms, and time. Backed by
browser `localStorage` with optional **Supabase cloud sync** and deployed on GitHub Pages
(`https://alanjmoses.github.io/work-tracker/`).

This file is the **index**. The spec lives in focused docs under [`docs/`](./docs/):

| Doc | Covers |
|---|---|
| [docs/product.md](./docs/product.md) | What it's for, every view (Focus / Tasks / Log / Timeline), the modals, interactions, out-of-scope |
| [docs/data-model.md](./docs/data-model.md) | The `feature` shape, holidays, work-blocks model, tags, status/priority semantics |
| [docs/architecture.md](./docs/architecture.md) | Single-file structure, data storage & cloud sync, no-data-loss rules, hosting/deploy |
| [docs/styling.md](./docs/styling.md) | Theme, colour tokens (source of truth), done-row tint, favicon |
| [docs/seed-data.md](./docs/seed-data.md) | Seeded history (Oct 2025→Jun 2026), FigJam import, PM-per-feature table |
| [docs/changelog.md](./docs/changelog.md) | Dated record of notable changes |

## Keeping docs in sync

These docs are the spec — **update the relevant `docs/` file in the same change as any code edit** so it
doesn't drift. Match the change to its doc: a sync/storage change → `architecture.md`; a colour/visual
change → `styling.md`; a view/behaviour change → `product.md`; a schema change → `data-model.md`. Add a
dated entry to `changelog.md` for anything notable.

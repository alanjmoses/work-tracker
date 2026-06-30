# Seed & Historical Data

One-time data seeded from Alan's FigJam roadmap board. These are reference/history; the live store is
described in [data-model.md](./data-model.md).

## Seed data (Oct 2025 → Jun 2026)

Full historical data seeded based on Alan's FigJam roadmap board. Covers:

- **Oct 2025:** Jiraaf App FD Details, Jiraaf Web FD Listing, Bond Analyser Revamp, RA Insights (start)
- **Nov 2025:** altGraaf FD Listing & Details, Unit Selection Improvements, Pending Orders
- **Dec 2025:** RA Insights (hi-fi), Opportunity Details Redesign
- **Jan 2026:** RA Insights Details Page, IFA Investment Activity, KYC Flow DOB Addition
- **Feb 2026:** Header Revamp, Form 15G/H, Profile Edit
- **Mar 2026:** HUF KYC Flow, Income Certificate, Form 121, Figma File Restructuring
- **Apr 2026:** FTI Homepage Redesign, Disbursal Flow Admin, Trust Markers Phase 1, Design System Structuring
- **May 2026:** Accessibility Audit, Trust Markers Phase 2, Audit Requirements FATCA
- **Jun 2026:** KYC Not-Done Users Homepage, Opportunity Details Redesign v2

## FigJam import (one-time)

`importFigjamOnce()` (flag `wt_figjam_import_v1`) seeds Alan's real history transcribed from the FigJam
roadmap (Oct 2025 → Jun 2026):

- **~106 daily notes** mapped to features by name, on the first day of each task span (overwrites existing
  notes on those dates).
- **7 new features** for stickies with no matching feature: Create Tab view for Sell, Location required
  messaging, App API Loading error states, APD Interviews (hiring), Internal Presentation, Automating
  Internal Presentation Design, Bond Baskets.
- **10 high-confidence holidays** (public holidays + clearly-marked single days off). Ambiguous light-grey
  stretches were left out — Alan marks those via the Day modal's Holiday tab.

Notes were transcribed from screenshots (best-effort); a few tiny sub-labels may need correction in-app.

## PM per feature (from FigJam workstreams)

Each feature's PM, inferred from its FigJam workstream. PMs: Ganesh, Pranchal, Vaishali, Rishu. **Wired
into the data model** as `feature.pm` (string | null); `backfillPMsOnce()` (flag `wt_pm_backfill_v1`) seeds
this onto existing features by name on first load. Editable from the Add/Edit Feature modal.

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

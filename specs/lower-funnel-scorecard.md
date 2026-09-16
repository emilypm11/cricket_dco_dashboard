# Spec: Lower Funnel Scorecard

Status: Implemented
Related file: `display-dashboard.html` (Lower Funnel Scorecard tab —
`lfState`, `lfDecisionFor`, `lfIsLowerFunnel`, `renderLowerFunnelScorecard`;
shared scoring pipeline change in `psScoreRows`)

## Problem

`specs/performance-scoring.md` named a Keep / Keep+Refresh / Monitor /
Rotate Out / Reduce / Cut / Deprioritize / Refresh Now recommendation
vocabulary as a Phase 2 idea with zero methodology — no thresholds, no
scope. The reference PCE dashboard's Lower Funnel Scorecard collapses
that down to three buckets (Keep / Rotate+Reduce / Cut) with summary
tiles (Creatives, Impressions, Keep, Rotate/Reduce, Cut) and a full,
sortable per-creative table.

## Decisions confirmed with Emily

- **Decision thresholds**: Composite Score ≥50 → Keep, 25–49 → Rotate /
  Reduce, <25 → Cut. Chosen over an arbitrary tercile split because it
  reuses the score bands the reference dashboard's own Methodology tab
  already documents (≥75 top quartile, 50–74 above median, 25–49 below
  median, <25 bottom quartile) — median-and-above stays, below-median
  gets rotated, bottom quartile gets cut. A creative with a null
  Composite Score gets no decision (shown as "—"), never silently
  defaulted to Cut.
- **Funnel scope**: rows whose `Funnel` value contains "lower"
  (case-insensitive) by default — matches "Lower Funnel", "Lower Funnel
  (eComm)", "Lower Funnel (DTC)", etc. A manual dropdown override lets
  Emily pick one exact Funnel value instead, for datasets that don't
  literally say "lower" (e.g. "BOFU").
- **Zero-LPV creatives**: per the reference Methodology screenshot, a
  creative with 0 Landing Page Visits gets its CVR and LPVR percentile
  components set to a neutral 50 rather than left null/at a real (often
  misleadingly low) rank — CVR is mathematically undefined at 0 LPVs, and
  a real 0% LPVR would otherwise sink a creative to the bottom of its
  peer group for what's frequently a tracking gap rather than bad
  performance. This changed the **shared** `psScoreRows` pipeline (not
  just this tab) — a zero-LPV creative's Composite Score on the
  Performance Data tab also changed from "N/A" to a real score, driven
  by its TDOR/CTR percentiles. Flagged with a `*` (tooltip) everywhere it
  appears so it's never mistaken for a normal score.

## Scope

- Independent state (`lfState`) from Performance Data and Benchmarks —
  its own dataset-type selector, date range, and filters (Publisher, Ad
  Type, Decision, creative-ID search).
- Summary tiles (Creatives / Impressions / Keep / Rotate+Reduce / Cut)
  reflect *all* lower-funnel rows regardless of the table's current
  filters, so they read as the whole scorecard's state rather than
  whatever narrower slice is currently displayed.
- Table columns trimmed from the reference's full set to the ones this
  dataset can actually support without fabricating data: Creative ID,
  Publisher, Funnel, Ad Type, Impressions, CTR, LPVs, LPVR, Orders, CVR,
  TDOR, Score (with a mini bar), Peer Rank, Decision. No Fatigue column —
  that's the separate, not-yet-spec'd Fatigue Monitor. No creative preview
  thumbnail — Cloudflare image matching is keyed to BigQuery's Creative
  Offer Name/Version, not to the crosstab's arbitrary Creative ID, so no
  thumbnail can be resolved for these rows without new matching logic.
- Sourced entirely from the uploaded crosstab (`PS_DATASETS`) — same as
  Performance Data and Benchmarks, no BigQuery involvement. Shows an
  upload prompt until a crosstab is loaded, and a distinct "nothing
  matched 'lower funnel' automatically" message (pointing at the scope
  dropdown) when a crosstab is loaded but the funnel filter matches
  nothing, so the two empty states aren't confused with each other.

## Verification

`psScoreRows` (with the zero-LPV change) and `lfDecisionFor` were run
together via `osascript -l JavaScript` against a hand-built 4-creative
peer group (one strong all-around, one zero-LPV creative with decent
TDOR/CTR, one weak-everywhere, one middling) before wiring anything to
the DOM. Composite scores were hand-verified metric-by-metric
(percentile ranks, the 0.35/0.30/0.20/0.15 weighting, and the zero-LPV
creative's neutral 0.5 cvr/lpvr components) and matched exactly;
resulting decisions (Keep/Rotate/Cut) and peer rank order were both
correct.
